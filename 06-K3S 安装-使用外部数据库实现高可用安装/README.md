Video Reference：[06-08 K3s 高可用安装](https://www.bilibili.com/video/BV1c3411Y7mM/?spm_id_from=333.788&vd_source=9560c118fae1db9638f05a6ba2527085)

GitHub README.md：

[06-安装-使用外部数据库实现高可用安装](https://github.com/kingsd041/k3s-tutorial/tree/main/06-安装-使用外部数据库实现高可用安装)

[07-安装-嵌入式DB的高可用](https://github.com/kingsd041/k3s-tutorial/tree/main/07-安装-嵌入式DB的高可用)

[08-安装-集群数据存储选项](https://github.com/kingsd041/k3s-tutorial/tree/main/08-安装-集群数据存储选项)

# 1 K3s 安装 - 使用外部数据库实现高可用安装

Reference：https://docs.rancher.cn/docs/k3s/installation/ha/_index

本节介绍了如何使用外部数据库安装一个高可用的 K3s 集群。

## 1.1 要求

单节点 k3s server 集群可以满足各种用例，但是对于需要 Kubernetes control-plane 稳定运行的重要环境，您可以在 HA 配置中运行 K3s。一个 K3s HA 集群由以下几个部分组成：

- 两个或多个server 节点，将为 Kubernetes API 提供服务并运行其他 control-plane 服务。
- 零个或多个agent 节点，用于运行您的应用和服务。
- 外部数据存储 (与单个 k3s server 设置中使用的嵌入式 SQLite 数据存储相反)
- 固定的注册地址，位于 server 节点的前面，以允许 agent 节点向集群注册

Agent 通过固定的注册地址进行注册，但注册后直接与其中一个 server 节点建立连接。这是一个由 k3s agent 进程发起的 websocket 连接，并由作为 agent 进程一部分运行的客户端负载均衡器维护。

## 1.2 架构

[![img](assets/1678159324545-079c1666-9ee3-4b1b-88e3-b5569bc2cfce.png)](https://camo.githubusercontent.com/7f5ccaaddb73822eea42c4c2e6dd09baa8d56425ebd1ec727d4d383efaa54f6f/687474703a2f2f646f63732e72616e636865722e636e2f6173736574732f696d616765732f6b33732d6172636869746563747572652d68612d7365727665722d34366266346333386532313032343662646135393230313237626265636435332e706e67)

## 1.3 安装

1. 环境介绍

| 主机名       | 角色       | IP            |
| ------------ | ---------- | ------------- |
| k3s-server-1 | k3s master | 10.0.0.30     |
| k3s-server-2 | k3s master | 10.0.0.31     |
| k3s-db       | DB         | 110.41.20.249 |
| k3s-lb       | LB         | 10.0.0.97     |
| k3s-agent    | k3s agent  | 10.0.0.32     |

1. 创建一个外部数据存储

你首先需要为集群创建一个外部数据存储。请参阅[集群数据存储选项](http://docs.rancher.cn/docs/k3s/installation/datastore/_index)文档了解更多细节。

K3s 支持以下数据存储选项：

- 嵌入式 SQLite
- PostgreSQL (经过认证的版本：10.7 和 11.5)
- MySQL (经过认证的版本：5.7)
- MariaDB (经过认证的版本：10.3.20)
- etcd (经过认证的版本：3.3.15)
- 嵌入式 etcd 高可用

```shell
# k3s-db
$ docker run --name some-mysql \
  --restart=unless-stopped \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=password \
  -d mysql:5.7

$ docker ps
CONTAINER ID   IMAGE             COMMAND                  CREATED          STATUS          PORTS                                                  NAMES
ad4fbe0ae143   mysql:5.7         "docker-entrypoint.s…"   16 seconds ago   Up 15 seconds   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp   some-mysql
```

1. 启动 k3s server 节点

```shell
# k3s-server-1
$ curl -sfL https://get.k3s.io | sh -s - server \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  --datastore-endpoint="mysql://root:password@tcp(110.41.20.249:3306)/k3s" \
	--tls-san 10.0.0.97
# INSTALL_K3S_EXEC="--cluster-init"
# 若出现下载过程失败，需要重复执行

# k3s-server-2
$ cat /var/lib/rancher/k3s/server/token
K10e4eb1b884d8a6259252b72ef3a508720976b3e87e0f7690ccc4c38a7b0a1e572::server:5ef7db1ffeba8f05653ad9e24d3b0764

$ curl -sfL https://get.k3s.io | sh -s - server \
  --token=5ef7db1ffeba8f05653ad9e24d3b0764 INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  --datastore-endpoint="mysql://username:password@tcp(110.41.20.249:3306)/k3s" \
  --tls-san 10.0.0.97

# 注意：应该备份 token 的值，因为从备份恢复和添加节点时都需要它。以前，K3s 在使用外部 SQL 数据存储时不强制使用 token。
```

💡国内用户，可以使用以下方法加速安装：

```shell
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -s - server \
  --token=SECRET \
  --datastore-endpoint="mysql://username:password@tcp(110.41.20.249:3306)/k3s" \
  --tls-san 10.0.0.97
```

--tls-san：在 TLS 证书中添加其他主机名或 IP 作为主题备用名称，本例为 LB 的 IP 否则通过 LB IP 连接 k3s api 时将会报错：`Unable to connect to the server: x509: certificate is valid for 10.43.0.1, 127.0.0.1, 172.31.2.134, 172.31.2.42, not 172.31.13.97`

```shell
root@k3s-server-1:~# kubectl get nodes
NAME           STATUS   ROLES                  AGE     VERSION
k3s-server-1   Ready    control-plane,master   7m15s   v1.20.7+k3s1
k3s-server-2   Ready    control-plane,master   3s      v1.20.7+k3s1

root@k3s-server-1:~# kubectl get pods -A
NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
kube-system   local-path-provisioner-5ff76fc89d-jk4tx   1/1     Running     0          7m9s
kube-system   metrics-server-86cbb8457f-9vmq9           1/1     Running     0          7m9s
kube-system   coredns-854c77959c-b7hvc                  1/1     Running     0          7m9s
kube-system   helm-install-traefik-jvvrs                0/1     Completed   0          7m9s
kube-system   svclb-traefik-2pg2w                       2/2     Running     0          6m53s
kube-system   traefik-6f9cbd9bd4-lg8vl                  1/1     Running     0          6m53s
kube-system   svclb-traefik-qdnjz                       2/2     Running     0          7s
```

默认情况下，server 节点将是可调度的，因此你的工作负载可以在它们上启动。如果你希望有一个专用的 control-plane，在这个平面上不会运行用户工作负载，你可以使用 taints。`node-taint` 参数将允许你用污点配置节点，例如`--node-taint CriticalAddonsOnly=true:NoExecute`。

1. 配置固定的注册地址

Agent 节点需要一个 URL 来注册，你应该在 server 节点前面有一个稳定的 endpoint，不会随时间推移而改变。可以使用许多方法来设置此 endpoint，例如：

- 一个 4 层（TCP）负载均衡器
- 轮询 DNS
- 虚拟或弹性 IP 地址

使用 nginx 作为负载均衡器，将 6443 端口流量转发到 k3s server:

```shell
cat install-docker.sh
#!/bin/bash

DOCKER_VERSION="20.10.10"
#UBUNTU_DOCKER_VERSION="5:${DOCKER_VERSION}~3-0~`lsb_release -si`-`lsb_release -cs`"
DOCKER_COMPOSE_VERSION=1.29.2
DOCKER_COMPOSE_FILE=docker-compose-Linux-x86_64

COLOR_SUCCESS="echo -e \\033[1;32m"
COLOR_FAILURE="echo -e \\033[1;31m"
END="\033[m"

. /etc/os-release

color () {
    RES_COL=60
    MOVE_TO_COL="echo -en \\033[${RES_COL}G"
    SETCOLOR_SUCCESS="echo -en \\033[1;32m"
    SETCOLOR_FAILURE="echo -en \\033[1;31m"
    SETCOLOR_WARNING="echo -en \\033[1;33m"
    SETCOLOR_NORMAL="echo -en \E[0m"
    echo -n "$1" && $MOVE_TO_COL
    echo -n "["
    if [ $2 = "success" -o $2 = "0" ] ;then
        ${SETCOLOR_SUCCESS}
        echo -n $"  OK  "
    elif [ $2 = "failure" -o $2 = "1"  ] ;then
        ${SETCOLOR_FAILURE}
        echo -n $"FAILED"
    else
        ${SETCOLOR_WARNING}
        echo -n $"WARNING"
    fi
    ${SETCOLOR_NORMAL}
    echo -n "]"
    echo
}


install_docker(){
    if [ $ID = "centos" -o $ID = "rocky" ];then
        if [ $VERSION_ID = "7" ];then
            cat >  /etc/yum.repos.d/docker.repo  <<EOF
[docker]
name=docker
gpgcheck=0
#baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable/
baseurl=https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/7/x86_64/stable/
EOF
        else
            cat >  /etc/yum.repos.d/docker.repo  <<EOF
[docker]
name=docker
gpgcheck=0
#baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/8/x86_64/stable/
baseurl=https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/8/x86_64/stable/
EOF
        fi
            yum clean all
        ${COLOR_FAILURE} "Docker有以下版本"${END}
        yum list docker-ce --showduplicates
        ${COLOR_FAILURE}"5秒后即将安装: docker-"${DOCKER_VERSION}" 版本....."${END}
        ${COLOR_FAILURE}"如果想安装其它Docker版本，请按ctrl+c键退出，修改版本再执行"${END}
        sleep 5
        yum -y install docker-ce-$DOCKER_VERSION docker-ce-cli-$DOCKER_VERSION  \
            || { color "Base,Extras的yum源失败,请检查yum源配置" 1;exit; }
    else
            dpkg -s docker-ce &> /dev/null && $COLOR"Docker已安装，退出" 1 && exit
        apt update || { color "更新包索引失败" 1 ; exit 1; }
        apt  -y install apt-transport-https ca-certificates curl software-properties-common || \
            { color "安装相关包失败" 1 ; exit 2;  }
        curl -fsSL https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
        add-apt-repository "deb [arch=amd64] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
        apt update
        ${COLOR_FAILURE} "Docker有以下版本"${END}
        apt-cache madison docker-ce
        ${COLOR_FAILURE}"5秒后即将安装: docker-"${UBUNTU_DOCKER_VERSION}" 版本....."${END}
        ${COLOR_FAILURE}"如果想安装其它Docker版本，请按ctrl+c键退出，修改版本再执行"${END}
        sleep 5
        apt -y  install docker-ce=${UBUNTU_DOCKER_VERSION} docker-ce-cli=${UBUNTU_DOCKER_VERSION}
    fi
    if [ $? -eq 0 ];then
        color "安装软件包成功"  0
    else
        color "安装软件包失败，请检查网络配置" 1
        exit
    fi

    mkdir -p /etc/docker
    tee /etc/docker/daemon.json <<-'EOF'
{
          "registry-mirrors": ["https://po13h3y1.mirror.aliyuncs.com","http://hub-mirror.c.163.com","https://mirror.ccs.tencentyun.com","http://f1361db2.m.daocloud.io"],
          "insecure-registries": ["harbor.kubesphere.com:80"]
}
EOF
    systemctl daemon-reload
    systemctl enable docker
    systemctl restart docker
    docker version && color "Docker 安装成功" 0 ||  color "Docker 安装失败" 1
        echo 'alias rmi="docker images -qa|xargs docker rmi -f"' >> ~/.bashrc
        echo 'alias rmc="docker ps -qa|xargs docker rm -f"' >> ~/.bashrc
}

install_docker_compose(){
    if [ $ID = "centos" -o $ID = "rocky" ];then
        ${COLOR_SUCCESS}"开始安装 Docker compose....."${END}
        sleep 1
        if [ ! -e  ${DOCKER_COMPOSE_FILE} ];then
            curl -L https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/${DOCKER_COMPOSE_FILE} -o /usr/bin/docker-compose
            # curl -L https://get.daocloud.io/docker/compose/releases/download/v2.12.2/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
        else
            mv ${DOCKER_COMPOSE_FILE} /usr/bin/docker-compose
        fi
        chmod +x /usr/bin/docker-compose
    else
        apt -y install docker-compose
    fi
    if docker-compose --version ;then
        ${COLOR_SUCCESS}"Docker Compose 安装完成"${END}
    else
        ${COLOR_FAILURE}"Docker compose 安装失败"${END}
        exit
    fi
}

install_docker
install_docker_compose
# k3s-lb 节点
cat >> /etc/nginx.conf <<EOF
worker_processes 4;
worker_rlimit_nofile 40000;

events {
    worker_connections 8192;
}

stream {
    upstream k3s_api {
        least_conn;
        server 10.0.0.30:6443 max_fails=3 fail_timeout=5s;
        server 10.0.0.31:6443 max_fails=3 fail_timeout=5s;
    }
    server {
        listen     6443;
        proxy_pass k3s_api;
    }
}
EOF
docker run -d --restart=unless-stopped \
  -p 6443:6443 \
  -v /etc/nginx.conf:/etc/nginx/nginx.conf \
  --name k3s-nginx-lb \
  --restart=Always \
  nginx:1.14 
```

1. 可选： 加入 Agent 节点

在 HA 集群中加入 agent 节点与在单个 server 集群中加入 agent 节点是一样的。你只需要指定 agent 应该注册到的 URL 和它应该使用的 token 即可。

```shell
# k3s-agent 节点
curl -sfL https://get.k3s.io | K3S_URL=https://172.31.13.97:6443 K3S_TOKEN=mynodetoken sh -
```

1. 通过 kubeconfig 访问 K3s 集群

此时，可以通过 LB IP 访问 k3s api:

```shell
kubectl get nodes
NAME           STATUS   ROLES                  AGE   VERSION
k3s-server-1   Ready    control-plane,master   68s   v1.20.7+k3s1
k3s-server-2   Ready    control-plane,master   66s   v1.20.7+k3s1
```

## 1.4 其他

如何在阿里云上搭建 K3s HA，可参考[这应该是最适合国内用户的 K3s HA 方案](https://mp.weixin.qq.com/s/0Wk2MzfWqMqt8DfUK_2ICA)