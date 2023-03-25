Video Reference：[06-08 K3s 高可用安装](https://www.bilibili.com/video/BV1c3411Y7mM/?spm_id_from=333.788&vd_source=9560c118fae1db9638f05a6ba2527085)

GitHub README.md：

[06-安装-使用外部数据库实现高可用安装](https://github.com/kingsd041/k3s-tutorial/tree/main/06-安装-使用外部数据库实现高可用安装)

[07-安装-嵌入式DB的高可用](https://github.com/kingsd041/k3s-tutorial/tree/main/07-安装-嵌入式DB的高可用)

[08-安装-集群数据存储选项](https://github.com/kingsd041/k3s-tutorial/tree/main/08-安装-集群数据存储选项)

# 3 集群数据存储选项

官方资料：https://docs.rancher.cn/docs/k3s/installation/datastore/_index

使用 `etcd` 以外的数据存储运行 `Kubernetes` 的能力使 `K3s` 区别于其他 `Kubernetes` 发行版。该功能为 `Kubernetes` 操作者提供了灵活性。可用的数据存储选项允许您选择一个最适合您用例的数据存储。例如：

- 如果你的团队没有操作 `etcd` 的专业知识，可以选择 `MySQL` 或 `PostgreSQL` 等企业级 SQL 数据库。
- 如果您需要在 `CI/CD` 环境中运行一个简单的、短暂的集群，您可以使用嵌入式 `SQLite` 数据库。
- 如果你希望在边缘部署 `Kubernetes`，并需要一个高可用的解决方案，但又无法承担在边缘管理数据库的操作开销，你可以使用 `K3s` 建立在嵌入式 `etcd` 之上的嵌入式 HA 数据存储。

K3s 支持以下数据存储选项：

- 嵌入式 [SQLite](https://www.sqlite.org/index.html)
- [PostgreSQL](https://www.postgresql.org/) (经过认证的版本：10.7 和 11.5)
- [MySQL](https://www.mysql.com/) (经过认证的版本：5.7)
- [MariaDB](https://mariadb.org/) (经过认证的版本：10.3.20)
- [etcd](https://etcd.io/) (经过认证的版本：3.3.15)
- 嵌入式 etcd 高可用

## 3.1 外部数据存储配置参数

如果你想使用外部数据存储，如 PostgreSQL、MySQL 或 etcd，你必须设置`datastore-endpoint`参数，以便 K3s 知道如何连接到它。你也可以指定参数来配置连接的认证和加密。下表总结了这些参数，它们可以作为 CLI 标志或环境变量传递。

| **CLI Flag**           | **环境变量**           | **描述**                                                     |
| ---------------------- | ---------------------- | ------------------------------------------------------------ |
| `--datastore-endpoint` | K3S_DATASTORE_ENDPOINT | 指定一个 PostgresSQL、MySQL 或 etcd 连接字符串。用于描述与数据存储的连接。这个字符串的结构是特定于每个后端的，详情如下。 |
| `--datastore-cafile`   | K3S_DATASTORE_CAFILE   | TLS 证书颁发机构（CA）文件，用于帮助确保与数据存储的通信安全。如果你的数据存储通过 TLS 服务请求，使用由自定义证书颁发机构签署的证书，你可以使用这个参数指定该 CA，这样 K3s 客户端就可以正确验证证书。 |
| `--datastore-certfile` | K3S_DATASTORE_CERTFILE | TLS 证书文件，用于对数据存储进行基于客户端证书的验证。要使用这个功能，你的数据存储必须被配置为支持基于客户端证书的认证。如果你指定了这个参数，你还必须指定datastore-keyfile参数。 |
| `--datastore-keyfile`  | K3S_DATASTORE_KEYFILE  | TLS 密钥文件，用于对数据存储进行基于客户端证书的认证。更多细节请参见前面的datastore-certfile参数。 |

作为最佳实践，我们建议将这些参数设置为环境变量，而不是命令行参数，这样你的数据库证书或其他敏感信息就不会作为进程信息的一部分暴露出来。

## 3.2 数据存储端点格式和功能[#](https://docs.rancher.cn/docs/k3s/installation/datastore/_index/#数据存储端点格式和功能)

如前所述，传递给datastore-endpoint参数的值的格式取决于数据存储后端。下文详细介绍了每个支持的外部数据存储的格式和功能。

#### 3.2.1 MySQL/MariaDB

最常见的 MySQL 和 MariaDB 的datastore-endpoint参数格式如下：

```shell
mysql://username:password@tcp(hostname:3306)/database-name
```

更多的高级配置参数，请参见 https://github.com/go-sql-driver/mysql#dsn-data-source-name

请注意，由于 K3s 中的一个[已知问题](https://github.com/rancher/k3s/issues/1093)，你无法设置`tls`参数。支持 TLS 通信，但你不能将这个参数设置为 "`skip-verify`"来使 K3s 跳过证书验证。

如果指定的数据库名称不存在，k3s server 将尝试创建它。

如果你只提供`mysql://`作为端点，K3s 将尝试做如下操作：

- 使用 root 用户并且不使用密码连接到`/var/run/mysqld/mysqld.sock`上的 MySQL 套接字
- 创建一个名为`kubernetes`的数据库

#### 3.2.2 etcd

最常见的 etcd 的`datastore-endpoint`参数的格式如下：

```shell
https://etcd-host-1:2379,https://etcd-host-2:2379,https://etcd-host-3:2379
```

以上假设是一个典型的三节点 etcd 集群。该参数可以再接受一个以逗号分隔的 etcd URL。

#### 3.2.3 PostgreSQL

最常见的 PostgreSQL 数据存储端点的参数格式：

```
postgres://username:password@hostname:port/database-name
```

更多的高级配置参数，请参见 https://godoc.org/github.com/lib/pq

如果指定的数据库名称不存在，k3s server 将尝试创建它。

如果你只提供`postgres://`作为端点，K3s 将尝试做如下操作：

- 使用`postgres`作为用户名和密码连接到 localhost
- 创建一个名为`kubernetes`的数据库

## 3.3 嵌入式 DB 的高可用[#](https://docs.rancher.cn/docs/k3s/installation/datastore/_index/#嵌入式-db-的高可用)

请参阅[嵌入式 DB 的高可用](https://docs.rancher.cn/docs/k3s/installation/ha-embedded/_index)了解如何使用该选项运行。
