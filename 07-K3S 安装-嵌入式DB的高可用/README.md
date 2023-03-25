Video Reference：[06-08 K3s 高可用安装](https://www.bilibili.com/video/BV1c3411Y7mM/?spm_id_from=333.788&vd_source=9560c118fae1db9638f05a6ba2527085)

GitHub README.md：

[06-安装-使用外部数据库实现高可用安装](https://github.com/kingsd041/k3s-tutorial/tree/main/06-安装-使用外部数据库实现高可用安装)

[07-安装-嵌入式DB的高可用](https://github.com/kingsd041/k3s-tutorial/tree/main/07-安装-嵌入式DB的高可用)

[08-安装-集群数据存储选项](https://github.com/kingsd041/k3s-tutorial/tree/main/08-安装-集群数据存储选项)

# 2 嵌入式 DB 的高可用安装

要在这种模式下运行 K3s，你必须有奇数的服务器节点。我们建议从三个节点开始。

要开始运行，首先启动一个服务器节点，使用 `cluster-init 标志来启用集群`，并使用一个标记作为共享的密钥来加入其他服务器到集群中。

```shell
K3S_TOKEN=SECRET k3s server --cluster-init
# 或者:
curl -sfL https://get.k3s.io | K3S_TOKEN=SECRET sh -s - --cluster-init

#################################################################################
# 示例：
# 服务器节点(启动etcd集群) (1/3)
# SECRET 我们预定一个 key 值
$ curl -sfL https://get.k3s.io | \
  INSTALL_K3S_MIRROR=cn K3S_TOKEN=SECRET \
  sh -s - --cluster-init

# 查看Token值
$ cat /var/lib/rancher/k3s/server/token
K10051c6b35b310436fdc52c0dac99e8ccbeb4bd16de3356c8a59c182e9cfddf791::server:SECRET

# 查看Node节点类型
# 默认使用嵌入式 etcd 数据库进行部署
$ sudo kubectl get nodes
NAME   STATUS   ROLES                       AGE     VERSION
k3s1   Ready    control-plane,etcd,master   2m20s   v1.25.7+k3s1
```

启动第一台服务器后，使用共享密钥将第二台和第三台服务器加入集群。

```shell
K3S_TOKEN=SECRET k3s server --server https://<ip or hostname of server1>:6443。
# 或者:
# 其他服务器节点(2/3)
# k3s-server-2 | k3s-server-3
$ curl -sfL https://get.k3s.io | \
  INSTALL_K3S_MIRROR=cn K3S_TOKEN=SECRET \
  sh -s - --server https://<ip-or-host-server>:6443

#################################################################################
# 示例：
$ curl -sfL https://get.k3s.io | \
  INSTALL_K3S_MIRROR=cn K3S_TOKEN=SECRET \
  sh -s - --server https://10.0.0.51:6443
# 使用etcd数据库启动K3s集群的时间要比使用SQLite数据库的K3s集群的时间要长

# 嵌入式的Etcd高可用集群搭建成功
$ kubectl get nodes
NAME   STATUS   ROLES                       AGE     VERSION
k3s1   Ready    control-plane,etcd,master   2m45s   v1.25.7+k3s1
k3s2   Ready    control-plane,etcd,master   91s     v1.25.7+k3s1
k3s3   Ready    control-plane,etcd,master   10s     v1.25.7+k3s1
```

查询 ETCD 集群状态：

```shell
# 查询ETCD集群状态
# etcd证书默认目录：/var/lib/rancher/k3s/server/tls/etcd
# etcd数据默认目录：/var/lib/rancher/k3s/server/db/etcd
ETCDCTL_ENDPOINTS='https://10.0.0.51:2379,https://10.0.0.52:2379,https://10.0.0.53:2379' \
ETCDCTL_CACERT='/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt' \
ETCDCTL_CERT='/var/lib/rancher/k3s/server/tls/etcd/server-client.crt' \
TCDCTL_KEY='/var/lib/rancher/k3s/server/tls/etcd/server-client.key' \
ETCDCTL_API=3 etcdctl endpoint status --write-out=table
```

- etcd 证书默认目录：`/var/lib/rancher/k3s/server/tls/etcd` 
- etcd 数据默认目录：`/var/lib/rancher/k3s/server/db/etcd`
