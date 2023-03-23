Video Reference：[06-08 K3s 高可用安装](https://www.bilibili.com/video/BV1c3411Y7mM/?spm_id_from=333.788&vd_source=9560c118fae1db9638f05a6ba2527085)

GitHub README.md：

[06-安装-使用外部数据库实现高可用安装](https://github.com/kingsd041/k3s-tutorial/tree/main/06-安装-使用外部数据库实现高可用安装)

[07-安装-嵌入式DB的高可用](https://github.com/kingsd041/k3s-tutorial/tree/main/07-安装-嵌入式DB的高可用)

[08-安装-集群数据存储选项](https://github.com/kingsd041/k3s-tutorial/tree/main/08-安装-集群数据存储选项)

# 2 嵌入式 DB 的高可用安装

要在这种模式下运行 K3s，你必须有奇数的服务器节点。我们建议从三个节点开始。

要开始运行，首先启动一个服务器节点，使用 cluster-init 标志来启用集群，并使用一个标记作为共享的密钥来加入其他服务器到集群中。

```shell
K3S_TOKEN=SECRET k3s server --cluster-init
或：
curl -sfL https://get.k3s.io | K3S_TOKEN=SECRET sh -s - --cluster-init
```

启动第一台服务器后，使用共享密钥将第二台和第三台服务器加入集群。

```shell
K3S_TOKEN=SECRET k3s server --server https://<ip or hostname of server1>:6443。
或:
curl -sfL https://get.k3s.io | K3S_TOKEN=SECRET sh -s - --server https://<ip or hostname of server1>:6443
```

查询 ETCD 集群状态：

```shell
ETCDCTL_ENDPOINTS='https://172.31.12.136:2379,https://172.31.4.43:2379,https://172.31.4.190:2379' ETCDCTL_CACERT='/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt' ETCDCTL_CERT='/var/lib/rancher/k3s/server/tls/etcd/server-client.crt' ETCDCTL_KEY='/var/lib/rancher/k3s/server/tls/etcd/server-client.key' ETCDCTL_API=3 etcdctl endpoint status --write-out=table
```

etcd 证书默认目录：`/var/lib/rancher/k3s/server/tls/etcd etcd` 数据默认目录：`/var/lib/rancher/k3s/server/db/etcd`