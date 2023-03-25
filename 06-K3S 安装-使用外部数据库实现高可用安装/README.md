Video Reference：[06-08 K3s 高可用安装](https://www.bilibili.com/video/BV1c3411Y7mM/?spm_id_from=333.788&vd_source=9560c118fae1db9638f05a6ba2527085)

GitHub README.md：

[06-安装-使用外部数据库实现高可用安装](https://github.com/kingsd041/k3s-tutorial/tree/main/06-安装-使用外部数据库实现高可用安装)

[07-安装-嵌入式DB的高可用](https://github.com/kingsd041/k3s-tutorial/tree/main/07-安装-嵌入式DB的高可用)

[08-安装-集群数据存储选项](https://github.com/kingsd041/k3s-tutorial/tree/main/08-安装-集群数据存储选项)

# 💎1 K3s 安装 - 使用外部数据库实现高可用安装

Reference：https://docs.rancher.cn/docs/k3s/installation/ha/_index

本节介绍了如何使用外部数据库安装一个高可用的 K3s 集群。

## 1.1 要求

单节点 k3s server 集群可以满足各种用例，但是对于需要 Kubernetes control-plane 稳定运行的重要环境，您可以在 HA 配置中运行 K3s。一个 K3s HA 集群由以下几个部分组成：

- 两个或多个server 节点，将为 Kubernetes API 提供服务并运行其他 control-plane 服务。
- 零个或多个agent 节点，用于运行您的应用和服务。
- 外部数据存储 (与单个 k3s server 设置中使用的嵌入式 SQLite 数据存储相反，内置的 SQLite 则不会起作用)
- 固定的注册地址，位于 server 节点的前面，以允许 agent 节点向集群注册[ 需要Kubectl连接到一个固定的注册地址 ]
- [这应该是最适合国内用户的 K3s HA 方案](https://mp.weixin.qq.com/s/0Wk2MzfWqMqt8DfUK_2ICA)

Agent 通过固定的注册地址进行注册，但注册后直接与其中一个 server 节点建立连接。这是一个由 k3s agent 进程发起的 websocket 连接，并由作为 agent 进程一部分运行的客户端负载均衡器维护。

## 1.2 架构

虽然单节点 `k3s server` 集群可以满足各种用例，但是对于需要稳定运行的重要环境，可以在 `HA` 配置中运行 `K3s`，如何使用外部数据库安装一个高可用的 `K3s` 集群？

[![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1678159324545-079c1666-9ee3-4b1b-88e3-b5569bc2cfce.png)](https://camo.githubusercontent.com/7f5ccaaddb73822eea42c4c2e6dd09baa8d56425ebd1ec727d4d383efaa54f6f/687474703a2f2f646f63732e72616e636865722e636e2f6173736574732f696d616765732f6b33732d6172636869746563747572652d68612d7365727665722d34366266346333386532313032343662646135393230313237626265636435332e706e67)

对于 Kubernetes control-plane 的正常运行至关重要的环境，您可以在高可用配置中运行 K3s。一个高可用 K3s 集群由以下几个部分组成：

- **K3s Server 节点**：两个或更多的server节点将为 Kubernetes API 提供服务并运行其他 control-plane 服务
- **外部数据库**：与单节点 k3s 设置中使用的嵌入式 SQLite 数据存储相反，高可用 K3s 需要挂载一个external database外部数据库作为数据存储的媒介。
- K3s高可用架构与K3s单节点架构主要的区别是高可用架构有一个外置的数据库，外置的数据库启动之后，在K3s Server通过配置连接到外置的数据库，当然K3s Server需要拥有多个，可以将 K3s Server理解为无状态的应用，因为K3s Server的数据都是存储在外置数据库中。
- 在前置需要有一个负载均衡器 LB。通过 LB 可以把对应的 k3s API 的流量分别转发到对应的 K3s Server 节点中。[ 另一种方式就是管理员直接调用某一台K3s Server 的 API，但是会出现K3s Server宕机的情况，会需要修改配置把K3s Server进行切换 ]
- 每个 K3s Agent 把 Agent 节点注册到K3s Server中。

## 1.3 安装

1. 环境介绍

| **主机名**   | **角色**   | **IP**        |
| ------------ | ---------- | ------------- |
| k3s-server-1 | k3s master | 10.0.0.51     |
| k3s-server-2 | k3s master | 10.0.0.52     |
| k3s-db       | DB         | 110.41.20.249 |
| k3s-lb       | LB         | 10.0.0.254    |
| k3s-agent    | k3s agent  | 10.0.0.53     |

1. 创建一个外部数据存储

你首先需要为集群创建一个外部数据存储。请参阅[集群数据存储选项](http://docs.rancher.cn/docs/k3s/installation/datastore/_index)文档了解更多细节。

K3s 支持以下数据存储选项[ 管理员对哪个数据库运维维护熟悉，就使用哪个数据库 ]：

- 嵌入式 SQLite
- PostgreSQL (经过认证的版本：10.7 和 11.5)
- MySQL (经过认证的版本：5.7)
- MariaDB (经过认证的版本：10.3.20)
- etcd (经过认证的版本：3.3.15)
- 嵌入式 etcd 高可用

```shell
# k3s-db
# 1.创建一个外部数据存储
$ docker run --name some-mysql \
  --restart=unless-stopped \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=password -e MYSQL_DATABASE=k3s \
  -e MYSQL_USER=rancher -e MYSQL_PASSWORD=password \
  -d mysql:5.7

$ docker ps
CONTAINER ID   IMAGE             COMMAND                  CREATED          STATUS          PORTS                                                  NAMES
ad4fbe0ae143   mysql:5.7         "docker-entrypoint.s…"   16 seconds ago   Up 15 seconds   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp   some-mysql
# 查看k3s数据库是否创建成功
$ docker exec -it some-mysql bash
root@4b8d836a6178:/# mysql -uroot -p
Enter password: 
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| k3s                |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)
```

1. 启动 k3s server 节点

```shell
# k3s-server-1 | k3s-server-2 节点执行
# 2.启动k3s-server节点(有读写权限不用加库名)
### mysql://username:password@tcp(hostname:3306)/database-name

# $ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
#   INSTALL_K3S_MIRROR=cn \
#   INSTALL_K3S_VERSION="v1.21.14+k3s1" \
#   sh -s - server \
#   --datastore-endpoint="mysql://root:password@tcp(110.41.20.249:3306)/k3s" \
#   --docker \
#   --tls-san 10.0.0.254(Failed)

####################################################################################
# k3s-server-1 | k3s-server-2 节点执行
# 注意：应该备份 token 的值，因为从备份恢复和添加节点时都需要它。以前，K3s 在使用外部 SQL 数据存储时不强制使用 token。
# k3s-server-1
$ curl -sfL https://get.k3s.io | sh -s - server \
  --datastore-endpoint="mysql://root:password@tcp(110.41.20.249:3306)/k3s" --tls-san 10.0.0.254
# 查看Token信息
$ cat /var/lib/rancher/k3s/server/token
K10362f77c8f9a55167f32e288e1cd0b234e37e225e4cb945d1303dd668b2c68d8a::server:ab3120b02be9f30b1acb98e25b84cea1

# k3s-server-2
$ curl -sfL https://get.k3s.io | sh -s - server \
  --token='ab3120b02be9f30b1acb98e25b84cea1' \
  --datastore-endpoint="mysql://root:password@tcp(110.41.20.249:3306)/k3s" --tls-san 10.0.0.254
```

💡国内用户，可以使用以下方法加速安装：

```shell
# INSTALL_K3S_MIRROR=cn 国内加速
$ curl -sfL https://get.k3s.io | INSTALL_K3S_MIRROR=cn \
  sh -s - server \
  --datastore-endpoint="mysql://root:password@tcp(110.41.20.249:3306)/k3s" --tls-san 10.0.0.254
--tls-san`：在 TLS 证书中添加其他主机名或 IP 作为主题备用名称，本例为 LB 的 IP 否则通过 LB IP 连接 k3s api 时将会报错：`Unable to connect to the server: x509: certificate is valid for 10.43.0.1, 127.0.0.1, 172.31.2.134, 172.31.2.42, not 172.31.13.97
# 查看节点情况
$ kubectl get nodes
NAME   STATUS   ROLES                  AGE   VERSION
k3s1   Ready    control-plane,master   23m   v1.25.7+k3s1
k3s2   Ready    control-plane,master   58s   v1.25.7+k3s1

# 查看Pod的资源
$ kubectl get pod -A
NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
kube-system   coredns-597584b69b-9p4tt                  1/1     Running     0          23m
kube-system   local-path-provisioner-79f67d76f8-rtf86   1/1     Running     0          23m
kube-system   helm-install-traefik-crd-qxgcb            0/1     Completed   0          23m
kube-system   helm-install-traefik-dks57                0/1     Completed   1          23m
kube-system   metrics-server-5f9f776df5-nzrsp           1/1     Running     0          23m
kube-system   svclb-traefik-faa2405d-qsh8n              2/2     Running     0          22m
kube-system   traefik-66c46d954f-b29lk                  1/1     Running     0          22m
kube-system   svclb-traefik-faa2405d-8ktrt              2/2     Running     0          62s
```

默认情况下，server 节点将是可调度的，因此你的工作负载可以在它们上启动。如果你希望有一个专用的 control-plane，在这个平面上不会运行用户工作负载，你可以使用 taints。`node-taint` 参数将允许你用污点配置节点，例如`--node-taint CriticalAddonsOnly=true:NoExecute`。

1. 配置固定的注册地址

Agent 节点需要一个 URL 来注册，你应该在 server 节点前面有一个稳定的 endpoint，不会随时间推移而改变。可以使用许多方法来设置此 endpoint，例如：

- 一个 4 层（TCP）负载均衡器
- 轮询 DNS
- 虚拟或弹性 IP 地址

使用 nginx 作为负载均衡器，将 6443 端口流量转发到 k3s server:

```shell
# 安装Docker
curl https://releases.rancher.com/install-docker/19.03.sh | sh
# k3s-lb 节点
# 3.配置固定的注册地址(k3s-lb节点)
# Agent节点需要一个URL来注册(LB)
cat >> /etc/nginx.conf <<EOF
worker_processes 4;
worker_rlimit_nofile 40000;

events {
    worker_connections 8192;
}

stream {
    upstream k3s_api {
        least_conn;
        server 10.0.0.51:6443 max_fails=3 fail_timeout=5s;
        server 10.0.0.52:6443 max_fails=3 fail_timeout=5s;
    }
    server {
        listen     6443;
        proxy_pass k3s_api;
    }
}
EOF
# 启动Nginx容器
$ docker run -d --restart=unless-stopped \
  -p 6443:6443 \
  -v /etc/nginx.conf:/etc/nginx/nginx.conf \
  --name k3s-nginx-lb \
  --restart=always \
  nginx:1.14

# 查看Nginx容器的状态
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                            NAMES
041803fe60e6        nginx:1.14          "nginx -g 'daemon of…"   6 seconds ago       Up 5 seconds        80/tcp, 0.0.0.0:6443->6443/tcp   k3s-nginx-lb

# 查看Nginx的日志
$ docker logs -f k3s-nginx-lb
```

1. 可选： 加入 Agent 节点

在 HA 集群中加入 agent 节点与在单个 server 集群中加入 agent 节点是一样的。你只需要指定 agent 应该注册到的 URL 和它应该使用的 token 即可。

```shell
# k3s-agent 节点
curl -sfL https://get.k3s.io | K3S_URL=https://[K3S_Load_Balancer]:6443 K3S_TOKEN=mynodetoken sh -

# 示例：
# 查看K3s Server集群的Token
$ cat /var/lib/rancher/k3s/server/token
K10362f77c8f9a55167f32e288e1cd0b234e37e225e4cb945d1303dd668b2c68d8a::server:ab3120b02be9f30b1acb98e25b84cea1

# 4.加入Agent节点
# Agent会保存LB节点和每个Server节点的IP信息
# cat /var/lib/rancher/k3s/agent/etc/k3s-agent-load-balancer.json
$ curl -sfL https://get.k3s.io | K3S_URL=https://10.0.0.254:6443 K3S_TOKEN=ab3120b02be9f30b1acb98e25b84cea1 sh -
# 只要LB后端有K3s Server存活并且可以处理请求，那么LB的流量就会打入到K3s Server
```

1. 通过 kubeconfig 访问 K3s 集群

此时，可以通过 LB IP 访问 k3s API：

```shell
# 查看节点情况
$ kubectl get nodes
NAME        STATUS   ROLES                  AGE   VERSION
k3s2        Ready    control-plane,master   24m   v1.25.7+k3s1
k3s1        Ready    control-plane,master   47m   v1.25.7+k3s1
k3s-agent   Ready    <none>                 15s   v1.25.7+k3s1

# 查看Pod的情况
$ kubectl get pod -A -o wide
NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE   IP          NODE        NOMINATED NODE   READINESS GATES
kube-system   coredns-597584b69b-9p4tt                  1/1     Running     0          47m   10.42.0.4   k3s1        <none>           <none>
kube-system   local-path-provisioner-79f67d76f8-rtf86   1/1     Running     0          47m   10.42.0.3   k3s1        <none>           <none>
kube-system   helm-install-traefik-crd-qxgcb            0/1     Completed   0          47m   10.42.0.2   k3s1        <none>           <none>
kube-system   helm-install-traefik-dks57                0/1     Completed   1          47m   10.42.0.5   k3s1        <none>           <none>
kube-system   metrics-server-5f9f776df5-nzrsp           1/1     Running     0          47m   10.42.0.6   k3s1        <none>           <none>
kube-system   svclb-traefik-faa2405d-qsh8n              2/2     Running     0          46m   10.42.0.7   k3s1        <none>           <none>
kube-system   traefik-66c46d954f-b29lk                  1/1     Running     0          46m   10.42.0.8   k3s1        <none>           <none>
kube-system   svclb-traefik-faa2405d-8ktrt              2/2     Running     0          25m   10.42.1.2   k3s2        <none>           <none>
kube-system   svclb-traefik-faa2405d-77qx6              2/2     Running     0          28s   10.42.2.2   k3s-agent   <none>           <none>
```

## 1.4 其他(阿里云上搭建 K3s HA)

如何在阿里云上搭建 K3s HA，可参考[这应该是最适合国内用户的 K3s HA 方案](https://mp.weixin.qq.com/s/0Wk2MzfWqMqt8DfUK_2ICA)