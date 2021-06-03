# 数据湖安全

## 概览

- 按层次划分
  - 湖体（S3）安全
  - 工具安全
  - 界面安全
- 按数据位置划分
  - 存储安全
  - 传输安全
- 安全的层面
  - 访问控制
  - 丢失和失窃
  - 防止数据篡改
  - 数据变更溯源
  - 数据监管要求（GDPR）
- 安全措施
  - 中心化权限管理
  - 联合身份认证
  - 基于角色的控制
  - 事前预防
  - 事中监控
  - 事后审计
  - 端到端保护

## 权责分工

- IAM
	- 负责运维相关的权限控制
	- 集群的控制
- Lake Formation
	- 负责数据湖的控制
- Redshift
	- 负责数据库和表的控制
	- 用户无需 IAM 账号，但需要能连接到集群
- Athena
	- 可以使用 IAM 账号权限控制
	- 额外有 Workshop 来做细粒度 IAM 限制
	- 可以跟 Lake Formation 结合，使用 ODBC + SAML 的方式来做验证
- VPC
	- 负责数据库直连 IP 的控制和审计


使用湖仓一体的方式，所有的访问都通过 Redshift，那么仅使用 Redshift 内部的
权限控制即可。用户不需要拥有 IAM 账号，只需要在 Redshift 中创建数据库用户并设置好细粒度权限即可。

对于在 Spectrum 上的数据也可以按照这个方式来进行权限的配置，但有一个问题，就是默认 Redshift 只能到 Schema 级别，而不能到表和字段级别。如果要配置 Lake Formation，则需要用到 IAM 角色。

也就是说，如果管理的粒度要到表、字段级别，就必须借助 Lake Formation 和 IAM 角色。这又带来另一个问题，那就是一个 Redshift 集群可以关联多个 IAM 角色，这些角色默认对所有集群内的用户开放。我们必须做一些限制。

用下面的语句，我们可以限制某个角色只能被某个集群中的某个用户所换用。

```
{
  "Version": "2012-10-17",
  "Statement": [
  {
    "Effect": "Allow",
    "Principal": { 
      "Service": "redshift.amazonaws.com" 
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "sts:ExternalId": [
          "arn:aws:redshift:us-west-2:123456789012:dbuser:my-cluster/user1",
          "arn:aws:redshift:us-west-2:123456789012:dbuser:my-cluster/user2"
        ]
      }
    }
  }]
}
```

也可以限制某个角色只能给某个区域的 Redshift 使用，进一步限制这个角色的使用范围。

这样做的问题有两个：

- 这些权限只支持 CREATE EXTERNAL SCHEMA 以及 COPY 等动作，只有这几个特定动作才支持在执行时附加 IAM 角色；对于 SELECT 等语句，这个 IAM 角色并没有意义
- 如果我们使用 GRANT XXXX ON SCHEMA 的方式，则又会碰到另一个问题，那就是 Redshift 在外部 SCHEMA 上只支持一个 USAGE 权限，而这个权限会放开外部 SCHEMA 下所有表的访问权限，无法做到细粒度限制

所以，我们还是需要把权限完整交给 Lake Formation。

这里有个巨大的坑，就是 Lake Formation 的工作原理，其实是会拿用户的角色，然后再去调用 GetDataAccess 拿到新的临时凭证，然后再进行查询。所以，使用 Lake Formation 的角色都必须有 `lakeformation:GetDataAccess` 的权限。如果还想兼容原来 Glue 的那一套 `GetTables` 权限，也可以加上，并且开启 Lake Formation 默认的 IAM 兼容设置。不过这个不推荐，因为相当于就绕过 Lake Formation 了。

创建外部数据库（`CREATE EXTERNAL SCHEMA ... FROM DATA CATALOG DATABASE`），其实用户并不需要对 Glue Database 本身的任何权限，只需要有这个数据库下的表的 `DESCRIBE` 权限即可。

这时候用户只有创建外部 SCHEMA 和外部表的权限，还没有访问外部数据的权限。如果我们尝试 `SELECT` 就会发现没有权限。

接下来我们需要添加表相关的权限。在这个过程中我们可以选择所有表，也可以选择单个表，还可以选择部分字段。

在选择部分字段之后，我们从 Redshift 里面查询的时候，就只能查询并且只会返回我们选择的字段，以及分区的字段。

是否支持 MySQL？不支持。你可能会觉得，MySQL 也可以被 Glue 爬虫爬下来并且保存进入 Glue DC，那么是不是 Redshift 还能查 MySQL 的数据，而且还可以用 Lake Formation 来限制访问。

这个想法很美好，但无法实现。Glue DC 中的 MySQL 表可以作为外部表纳入到 Redshift，但是却无法查询，查询时会报错。

当然，在 Redshift 中访问数据库的需求也并不是不存在。比如有时候我们可能会需要在统计的过程中查询一下用户的名字之类的数据，这时候就需要联合数据湖、仓库和数据库中的数据。

为了解决这个问题，Redshift 提出了 Federated Query 的概念，可以不通过 Lake Formation 和 Glue，直接把 MySQL 建立成外部表然后进行查询。

目前这个功能限制还比较多，比如只支持 RDS，必须使用 IAM Role 和 Secret Manager 等。

另外需要注意的是，Lake Formation 的 Data Location 权限仅仅用于限制用户创建指向该路径的数据库和表，并不妨碍用户对已经存在的表做查询和修改（`SELECT` 和 `INSERT`）。

此外，除了直接赋予用户 Data Location 显性权限之外，还有隐性权限。比如 Lake Formation 中已有数据库指向某个 S3 地址，然后用户拥有此数据库的 `CREATE_TABLE` 权限，则用户可以创建指向该 S3 地址的表。

Redshift 可以关联多个 Role。这些 Role 用于：

- 从 S3 上 COPY 数据
- UNLOAD 数据到 S3
- 创建外部表并从外部表查询数据

我们可以 GRANT ASSUMEROLE，但是这个 GRANT 仅仅只针对 COPY / UNLOAD 的操作，用户创建 EXTERNAL SCHEMA 的权力完全不受影响。创建完成后，用户即可以直接利用对应的 Role 来查询外部表，就绕过了 Role 的限制条件。

如果希望限制用户只能只用某个 Role，那么我们只能曲线救国。首先，禁用 Database 的 CREATE 权限，那么用户就不能创建 EXTERNAL SCHEMA；然后，由管理员创建好 EXTERNAL SCHEMA 并 GRANT 相应的权限给用户。

不同的用户，可以使用不同的 Role 来创建不同的 EXTERNAL SCHEMA，从而达到非常细粒度的权限控制。

我们可以简单理解成两个部分：

- 谁可以使用这个 Role
	- COPY / UNLOAD 操作
	- 创建外部数据集合的操作
- 这个 Role 有什么权限
	- Lake Formation 从数据库、表、行、列进行限制



















