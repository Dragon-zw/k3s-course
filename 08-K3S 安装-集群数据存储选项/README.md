Video Reference：[06-08 K3s 高可用安装](https://www.bilibili.com/video/BV1c3411Y7mM/?spm_id_from=333.788&vd_source=9560c118fae1db9638f05a6ba2527085)

GitHub README.md：

[06-安装-使用外部数据库实现高可用安装](https://github.com/kingsd041/k3s-tutorial/tree/main/06-安装-使用外部数据库实现高可用安装)

[07-安装-嵌入式DB的高可用](https://github.com/kingsd041/k3s-tutorial/tree/main/07-安装-嵌入式DB的高可用)

[08-安装-集群数据存储选项](https://github.com/kingsd041/k3s-tutorial/tree/main/08-安装-集群数据存储选项)

# 3 集群数据存储选项

使用 etcd 以外的数据存储运行 Kubernetes 的能力使 K3s 区别于其他 Kubernetes 发行版。该功能为 Kubernetes 操作者提供了灵活性。可用的数据存储选项允许您选择一个最适合您用例的数据存储。例如：

- 如果你的团队没有操作 etcd 的专业知识，可以选择 MySQL 或 PostgreSQL 等企业级 SQL 数据库。
- 如果您需要在 CI/CD 环境中运行一个简单的、短暂的集群，您可以使用嵌入式 SQLite 数据库。
- 如果你希望在边缘部署 Kubernetes，并需要一个高可用的解决方案，但又无法承担在边缘管理数据库的操作开销，你可以使用 K3s 建立在嵌入式 etcd 之上的嵌入式 HA 数据存储。

K3s 支持以下数据存储选项：

- 嵌入式 [SQLite](https://www.sqlite.org/index.html)
- [PostgreSQL](https://www.postgresql.org/) (经过认证的版本：10.7 和 11.5)
- [MySQL](https://www.mysql.com/) (经过认证的版本：5.7)
- [MariaDB](https://mariadb.org/) (经过认证的版本：10.3.20)
- [etcd](https://etcd.io/) (经过认证的版本：3.3.15)
- 嵌入式 etcd 高可用

## 3.1 外部数据存储配置参数

如果你想使用外部数据存储，如 PostgreSQL、MySQL 或 etcd，你必须设置`datastore-endpoint`参数，以便 K3s 知道如何连接到它。你也可以指定参数来配置连接的认证和加密。下表总结了这些参数，它们可以作为 CLI 标志或环境变量传递。

| **CLI Flag**         | **环境变量**           | **描述**                                                     |
| -------------------- | ---------------------- | ------------------------------------------------------------ |
| --datastore-endpoint | K3S_DATASTORE_ENDPOINT | 指定一个 PostgresSQL、MySQL 或 etcd 连接字符串。用于描述与数据存储的连接。这个字符串的结构是特定于每个后端的，详情如下。 |
| --datastore-cafile   | K3S_DATASTORE_CAFILE   | TLS 证书颁发机构（CA）文件，用于帮助确保与数据存储的通信安全。如果你的数据存储通过 TLS 服务请求，使用由自定义证书颁发机构签署的证书，你可以使用这个参数指定该 CA，这样 K3s 客户端就可以正确验证证书。 |
| --datastore-certfile | K3S_DATASTORE_CERTFILE | TLS 证书文件，用于对数据存储进行基于客户端证书的验证。要使用这个功能，你的数据存储必须被配置为支持基于客户端证书的认证。如果你指定了这个参数，你还必须指定datastore-keyfile参数。 |
| --datastore-keyfile  | K3S_DATASTORE_KEYFILE  | TLS 密钥文件，用于对数据存储进行基于客户端证书的认证。更多细节请参见前面的datastore-certfile参数。 |

作为最佳实践，我们建议将这些参数设置为环境变量，而不是命令行参数，这样你的数据库证书或其他敏感信息就不会作为进程信息的一部分暴露出来。