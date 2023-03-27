Video Reference：[13-备份和恢复](https://www.bilibili.com/video/BV1Dm4y1U7fw/?spm_id_from=pageDriver&vd_source=9560c118fae1db9638f05a6ba2527085)

GitHub Reference：[13-备份和恢复](https://github.com/kingsd041/k3s-tutorial/tree/main/13-备份和恢复)

**SQLite + etcd + 外部数据存储**

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

# 1 K3s 备份和恢复

K3s 的备份和恢复方式取决于使用的数据存储类型：

- 使用嵌入式 SQLite 数据存储进行备份和恢复
- 使用外部数据存储进行备份和恢复
- 使用嵌入式 etcd 数据存储进行备份和恢复

官方文档资料：https://docs.rancher.cn/docs/k3s/backup-restore/_index

## 1.1 配置基础环境

```shell
# 安装Docker容器运行时环境
# 在 K3s 节点上安装 Docker。可以使用 Rancher 的一个Docker 安装脚本来安装 Docker：
curl https://releases.rancher.com/install-docker/19.03.sh | sh

# 配置镜像加速器
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://po13h3y1.mirror.aliyuncs.com","http://hub-mirror.c.163.com","https://mirror.ccs.tencentyun.com","http://f1361db2.m.daocloud.io"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker && sudo systemctl enable docker
# K3s Master
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  INSTALL_K3S_EXEC="server --docker --node-name k3s1" \
  K3S_TOKEN=rancher sh -s -

$ cat /var/lib/rancher/k3s/server/token
K1046b15950a113b87d32b8d245b15490b76722b81f16b79f0e1eb912d2fe0d6b9a::server:rancher

# K3s Worker
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  K3S_URL=https://10.0.0.51:6443 \
  K3S_TOKEN=rancher INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  INSTALL_K3S_EXEC="--node-ip=10.0.0.52 --docker --node-name k3s2" \
  sh -

# 查看节点状态
$ kubectl get nodes -o wide
NAME   STATUS   ROLES                  AGE   VERSION         INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
k3s2   Ready    <none>                 13m   v1.21.14+k3s1   10.0.0.52     <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://19.3.15
k3s1   Ready    control-plane,master   15m   v1.21.14+k3s1   10.0.0.51     <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://19.3.15

# 创建测试Pod
$ kubectl create deployment nginx-deploy --image=nginx --replicas=3
$ kubectl get pod -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP          NODE   NOMINATED NODE   READINESS GATES
nginx-deploy-8588f9dfb-sf8p8   1/1     Running   0          61s   10.42.0.9   k3s1   <none>           <none>
nginx-deploy-8588f9dfb-4pgtx   1/1     Running   0          61s   10.42.1.4   k3s2   <none>           <none>
nginx-deploy-8588f9dfb-5xrzv   1/1     Running   0          61s   10.42.1.3   k3s2   <none>           <none>
```

## 1.2 💎使用嵌入式 SQLite 数据存储进行备份和恢复

### 1.2.1 方式1：备份/恢复数据目录

- 备份数据目录

```shell
$ ls -l /var/lib/rancher/k3s/server/db
total 11076
-rw-r--r-- 1 root root 2920448 Mar 24 21:03 state.db
-rw-r--r-- 1 root root   32768 Mar 24 21:03 state.db-shm
-rw-r--r-- 1 root root 4956392 Mar 24 21:04 state.db-wal

$ cp -av /var/lib/rancher/k3s/server/db /opt/db
‘/var/lib/rancher/k3s/server/db’ -> ‘/opt/db’
‘/var/lib/rancher/k3s/server/db/state.db’ -> ‘/opt/db/state.db’
‘/var/lib/rancher/k3s/server/db/state.db-wal’ -> ‘/opt/db/state.db-wal’
‘/var/lib/rancher/k3s/server/db/state.db-shm’ -> ‘/opt/db/state.db-shm’
```

- 恢复数据目录

```shell
# 删除Deployment的资源
$ kubectl delete deployment nginx-deploy
$ kubectl get pod -o wide 
No resources found in default namespace.

# 停止K3s服务并删除数据存储
# 其实K3s服务的停止和重启，并不会影响K3s集群中创建的业务Pod
$ systemctl stop k3s
$ rm -rf /var/lib/rancher/k3s/server/db

# 恢复数据目录
$ cp -av /opt/db /var/lib/rancher/k3s/server/db
# 重新启动k3s会加载SQLite的数据库
$ systemctl start k3s

# 查看Node的节点的状态
$ kubectl get nodes -o wide
NAME   STATUS   ROLES                  AGE   VERSION         INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
k3s2   Ready    <none>                 30m   v1.21.14+k3s1   10.0.0.52     <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://19.3.15
k3s1   Ready    control-plane,master   30m   v1.21.14+k3s1   10.0.0.51     <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://19.3.15

# 查看Pod恢复成功
$ kubectl get pod -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP           NODE   NOMINATED NODE   READINESS GATES
nginx-deploy-8588f9dfb-685f4   1/1     Running   0          92s   10.42.0.27   k3s1   <none>           <none>
nginx-deploy-8588f9dfb-sk2v6   1/1     Running   0          92s   10.42.0.28   k3s1   <none>           <none>
nginx-deploy-8588f9dfb-47rbf   1/1     Running   0          92s   10.42.0.29   k3s1   <none>           <none>
```

### 1.2.2 👑方式2：通过 SQLite cli

- 备份

```shell
# 使用sqlite3 CLI进行数据的备份
$ sqlite3 /var/lib/rancher/k3s/server/db/state.db
SQLite version 3.22.0 2018-01-22 18:45:57
Enter ".help" for usage hints.
sqlite> .backup "/opt/kine.db"
sqlite> .exit
```

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679663589509-dec71a64-80da-4189-b173-0b4a4e001646.png)

- 恢复

```shell
# 删除测试Pod并停止K3s服务
$ kubectl delete deployment nginx-deploy
$ systemctl stop k3s

# 使用sqlite3 CLI进行数据的恢复
$ sqlite3 /var/lib/rancher/k3s/server/db/state.db
SQLite version 3.22.0 2018-01-22 18:45:57
Enter ".help" for usage hints.
sqlite> .restore '/opt/kine.db'
sqlite> .exit

# 启动K3s服务
$ systemctl start k3s
# 查看Pod恢复成功
$ kubectl get pod
NAME                           READY   STATUS    RESTARTS   AGE
nginx-deploy-8588f9dfb-sk2v6   1/1     Running   0          6m8s
nginx-deploy-8588f9dfb-47rbf   1/1     Running   0          6m8s
nginx-deploy-8588f9dfb-685f4   1/1     Running   0          6m8s

# 注意K3s-agent是不会有数据目录的
# SQLite只有在K3s-server节点(或者K3s-master节点)存在数据目录
```

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679663662322-6da4758c-4e46-49ac-b904-5a495e986f95.png)

## 1.3 💎使用外部数据存储进行备份和恢复

当使用外部数据存储时，备份和恢复操作是在 K3s 之外处理的。数据库管理员需要对外部数据库进行备份，或者从快照或转储中进行恢复。我们建议将数据库配置为执行定期快照。

### 1.3.1 创建MySQL数据库

```shell
# 清理K3s集群信息
$ k3s-uninstall.sh && rm -rf /var/lib/rancher/ /etc/rancher/
$ k3s-agent-uninstall.sh && rm -rf /var/lib/rancher/ /etc/rancher/

$ mkdir -pv /mydata/mysql/conf/
$ cat <<-'EOF' > /mydata/mysql/conf/my.cnf
[client]
default-character-set=utf8mb4

[mysql]
default-character-set=utf8mb4

[mysqld]
init_connect='SET collation_connection = utf8mb4_unicode_ci'
init_connect='SET NAMES utf8mb4'
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci
skip-character-set-client-handshake
# 忽略表名的大小写
lower_case_table_names=1
# 跳过域名解析
skip-name-resolve
EOF

# 使用Docker启动一个数据库
$ docker run -p 3306:3306 --name mysql-rancher \
  -v /mydata/mysql/log:/var/log/mysql \
  -v /mydata/mysql/data:/var/lib/mysql \
  -v /mydata/mysql/conf:/etc/mysql/conf.d \
  -e MYSQL_ROOT_PASSWORD=kubesphere \
  -e MYSQL_USER=rancher \
  -e MYSQL_PASSWORD=kubesphere \
  -e MYSQL_DATABASE=k3s \
  --restart=always \
  -d mysql:5.7.36

# 查看MySQL数据库的状态
$ docker ps
CONTAINER ID        IMAGE                        COMMAND                  CREATED              STATUS              PORTS                               NAMES
3c49a4f950b6        mysql:5.7.36                 "docker-entrypoint.s…"   About a minute ago   Up About a minute   0.0.0.0:3306->3306/tcp, 33060/tcp   mysql-rancher
$ docker exec -it mysql-rancher /bin/bash
root@726d755c05b1:/# mysql -uroot -pkubesphere
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

### 1.3.2 K3s集群使用外部MySQL进行数据存储

```shell
# K3s Master
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  INSTALL_K3S_EXEC="server --docker --node-name k3s1" \
  K3S_TOKEN=rancher sh -s - server \
  --datastore-endpoint="mysql://root:kubesphere@tcp(10.0.0.51:3306)/k3s"

$ cat /var/lib/rancher/k3s/server/token
K103d8825f0905016b902c890103f80abbf1447a0768b0e9a3943d9d92cfd312736::server:rancher

# K3s Worker
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  K3S_URL=https://10.0.0.51:6443 \
  K3S_TOKEN=rancher INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  INSTALL_K3S_EXEC="--node-ip=10.0.0.52 --docker --node-name k3s2" \
  sh -s - server \
  --datastore-endpoint='mysql://root:kubesphere@tcp(10.0.0.51:3306)/k3s'

# 查看节点状态
$ kubectl get nodes  -o wide
NAME   STATUS   ROLES                  AGE   VERSION         INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
k3s1   Ready    control-plane,master   67s   v1.21.14+k3s1   10.0.0.51     <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://19.3.15
k3s2   Ready    <none>                 10s   v1.21.14+k3s1   10.0.0.52     <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://19.3.15

# 创建测试Pod
$ kubectl create deployment nginx-deploy --image=nginx --replicas=3
$ kubectl get pod -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP          NODE   NOMINATED NODE   READINESS GATES
nginx-deploy-8588f9dfb-lbsj5   1/1     Running   0          89s   10.42.0.9   k3s1   <none>           <none>
nginx-deploy-8588f9dfb-plmsw   1/1     Running   0          89s   10.42.1.4   k3s2   <none>           <none>
nginx-deploy-8588f9dfb-nvmtj   1/1     Running   0          89s   10.42.1.3   k3s2   <none>           <none>
```

### 1.3.3 验证数据备份和数据恢复

- 备份

```shell
# MySQL 备份数据库
# mysqldump -uroot -pkubesphere --all-databases --master-data > /opt/k3s-dbdump.db

# 进入到MySQL容器中
$ docker exec -it mysql-rancher bash
root@726d755c05b1:/# mysqldump -uroot -pkubesphere --all-databases > k3s-dbdump.db
mysqldump: [Warning] Using a password on the command line interface can be insecure.

# 拷贝到宿主机中
$ docker cp mysql-rancher:/k3s-dbdump.db /tmp
```

- 恢复

停止K3s服务

```shell
# 删除测试Pod
$ kubectl delete deployment nginx-deploy
# 停止K3s服务
$ systemctl stop k3s
```

恢复MySQL数据

```shell
# mysql -uroot -pkubesphere < k3s-dbdump.db 

# 进入到MySQL容器中
$ docker exec -it mysql-rancher bash
root@726d755c05b1:/# mysql -uroot -pkubesphere < k3s-dbdump.db 
mysqldump: [Warning] Using a password on the command line interface can be insecure.
```

启动K3s服务

```shell
# 启动K3s服务
$ systemctl start k3s

# 查看Pod的恢复状态
$ kubectl get pod -o wide
NAME                           READY   STATUS    RESTARTS   AGE     IP           NODE   NOMINATED NODE   READINESS GATES
nginx-deploy-8588f9dfb-s4nqk   1/1     Running   0          2m16s   10.42.1.3    k3s2   <none>           <none>
nginx-deploy-8588f9dfb-9gt5x   1/1     Running   0          2m16s   10.42.1.4    k3s2   <none>           <none>
nginx-deploy-8588f9dfb-mt896   1/1     Running   0          2m16s   10.42.0.10   k3s1   <none>           <none>
```

## 1.4 使用嵌入式 etcd 数据存储进行备份和恢复

### 1.4.1 搭建一套etcd高可用集群（可选择）

此处为语雀内容卡片，点击链接查看：https://www.yuque.com/seekerzw/xi8l23/epqc7oulvhpmbn9q#OogzL

下载证书工具

```shell
# 下载cfssl核心组件
wget https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssl-certinfo_1.5.0_linux_amd64
wget https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssl_1.5.0_linux_amd64
wget https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssljson_1.5.0_linux_amd64

# 授予执行权限
chmod +x cfssl*

# 批量重命名
for name in `ls cfssl*`; do mv $name ${name%_1.5.0_linux_amd64};  done

# 移动到文件
mv cfssl* /usr/bin

# 设置主机名
hostnamectl set-hostname k3s-master01
hostnamectl set-hostname k3s-master02
hostnamectl set-hostname k3s-master03
```

etcd 集群配置

```shell
# 给所有master节点，发送etcd包准备部署etcd高可用
wget https://github.com/etcd-io/etcd/releases/download/v3.4.16/etcd-v3.4.16-linux-amd64.tar.gz

cat <<-'EOF' >> /etc/hosts
10.0.0.51 k3s-master01
10.0.0.52 k3s-master02
10.0.0.53 k3s-master03
EOF

# 复制到其他节点
for i in k3s-master01 k3s-master02 k3s-master03;do scp etcd-* root@$i:/root/;done

# 解压到 /usr/local/bin
tar -zxvf etcd-v3.4.16-linux-amd64.tar.gz --strip-components=1 -C /usr/local/bin etcd-v3.4.16-linux-amd64/etcd{,ctl}
for i in k3s-master01 k3s-master02 k3s-master03; do ssh root@$i tar -zxvf etcd-v3.4.16-linux-amd64.tar.gz --strip-components=1 -C /usr/local/bin etcd-v3.4.16-linux-amd64/etcd{,ctl}; done

# /usr/local/bin 用户和用户组设置
chown root.root /usr/local/bin/etcd*

# 验证命令
etcdctl 
# 只要有打印就ok
```

\# 生成etcd证书

- **etcd-ca-csr.json**

```json
// etcd CA根机构证书

cat <<EOF > etcd-ca-csr.json
{
  "CN": "etcd",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "etcd",
      "OU": "etcd"
    }
  ],
  "ca": {
    "expiry": "87600h"
  }
}
EOF
# 生成etcd根ca证书
mkdir -pv /etc/rancher/pki/etcd/
cfssl gencert -initca etcd-ca-csr.json | cfssljson -bare /etc/rancher/pki/etcd/ca -

$ ls -l /etc/rancher/pki/etcd/
total 16
-rw-r--r-- 1 root root 1037 Jan 20 20:10 ca.csr			# 证书申请时的服务器响应文件，可以删除
-rw------- 1 root root 1679 Jan 20 20:10 ca-key.pem # 私钥
-rw-r--r-- 1 root root 1289 Jan 20 20:10 ca.pem     # 公钥
```

- **etcd-itdachang-csr.json**

```json
cat <<EOF > etcd-rancher-csr.json
{
  "CN": "etcd-rancher",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "hosts": [  
    "127.0.0.1",
    "k3s-master01",
    "k3s-master02",
    "k3s-master03",
    "10.0.0.51",
    "10.0.0.52",
    "10.0.0.53"
  ],
  "names": [
    {
      "C": "CN",
      "L": "beijing",
      "O": "etcd",
      "ST": "beijing",
      "OU": "System"
    }
  ]
}
EOF

// 生成etcd-itdachang-csr证书
// 注意：hosts用自己的主机名和ip
// 也可以在签发的时候再加上 -hostname=127.0.0.1,k3s-master01,k3s-master02,k3s-master03,
// 可以指定受信的主机列表
//    "hosts": [
  //        "k8s-master01",
  //        "www.example.net"
  //    ],
mkdir -p /etc/rancher/pki
cat <<-'EOF' > /etc/rancher/pki/ca-config.json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "server": {
        "expiry": "87600h",
        "usages": [
          "signing",
          "key encipherment",
          "server auth"
        ]
      },
      "client": {
        "expiry": "87600h",
        "usages": [
          "signing",
          "key encipherment",
          "client auth"
        ]
      },
      "peer": {
        "expiry": "87600h",
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ]
      },
      "kubernetes": {
        "expiry": "87600h",
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ]
      },
      "etcd": {
        "expiry": "87600h",
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ]
      }
    }
  }
}
EOF

# 签发itdachang的etcd证书
cfssl gencert \
    -ca=/etc/rancher/pki/etcd/ca.pem \
    -ca-key=/etc/rancher/pki/etcd/ca-key.pem \
    -config=/etc/rancher/pki/ca-config.json \
    -profile=etcd \
    etcd-rancher-csr.json | cfssljson -bare /etc/rancher/pki/etcd/etcd
# ca-config.json CA机构的配置，相当于定义一个多环境怎么工作

$ ls -l /etc/rancher/pki/etcd/etcd*
-rw-r--r-- 1 root root 1135 Jan 20 20:16 etcd.csr			# 证书申请时的服务器响应文件，可以删除
-rw------- 1 root root 1675 Jan 20 20:16 etcd-key.pem # 私钥
-rw-r--r-- 1 root root 1497 Jan 20 20:16 etcd.pem			# 公钥
#for i in k3s-master02 k3s-master03;do ssh root@$i mkdir -pv /etc/rancher/pki/etcd; done
#for i in k3s-master02 k3s-master03;do scp -r /etc/rancher/pki/etcd root@$i:/etc/rancher/pki; done
```

etcd配置信息

```shell
# 三个master执行
for i in k3s-master01 k3s-master02 k3s-master03;do ssh root@$i mkdir -pv /etc/etcd; done
# mkdir -p /etc/etcd
$ vim /etc/etcd/etcd.yaml
name: 'etcd-master02' # [etcd-master01 etcd-master02 etcd-master03]
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://10.0.0.52:2380' # [https://10.0.0.51:2380 https://10.0.0.52:2380 https://10.0.0.53:2380]
listen-client-urls: 'https://10.0.0.52:2379,http://127.0.0.1:2379' # [https://10.0.0.51:2379 https://10.0.0.52:2379 https://10.0.0.53:2379]
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://10.0.0.52:2380' # [https://10.0.0.51:2380 https://10.0.0.52:2380 https://10.0.0.53:2380]
advertise-client-urls: 'https://10.0.0.52:2379'				# [https://10.0.0.51:2379 https://10.0.0.52:2379 https://10.0.0.53:2379]
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'etcd-master01=https://10.0.0.51:2380,etcd-master02=https://10.0.0.52:2380,etcd-master03=https://10.0.0.53:2380'
initial-cluster-token: 'etcd-k8s-cluster'
initial-cluster-state: 'new'
strict-reconfig-check: false
enable-v2: true
enable-pprof: true
proxy: 'off'
proxy-failure-wait: 5000
proxy-refresh-interval: 30000
proxy-dial-timeout: 1000
proxy-write-timeout: 5000
proxy-read-timeout: 0
client-transport-security:
  cert-file: '/etc/rancher/pki/etcd/etcd.pem'
  key-file: '/etc/rancher/pki/etcd/etcd-key.pem'
  client-cert-auth: true
  trusted-ca-file: '/etc/rancher/pki/etcd/ca.pem'
  auto-tls: true
peer-transport-security:
  cert-file: '/etc/rancher/pki/etcd/etcd.pem'
  key-file: '/etc/rancher/pki/etcd/etcd-key.pem'
  peer-client-cert-auth: true
  trusted-ca-file: '/etc/rancher/pki/etcd/ca.pem'
  auto-tls: true
debug: false
log-package-levels:
log-outputs: [default]
force-new-cluster: false
```

启动服务

```shell
cat <<-'EOF' > /usr/lib/systemd/system/etcd.service

[Unit]
Description=Etcd Service
Documentation=https://etcd.io/docs/v3.4/op-guide/clustering/
After=network.target

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd --config-file=/etc/etcd/etcd.yaml
Restart=on-failure
RestartSec=10
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
Alias=etcd3.service
# 服务别名
EOF

# 加载&开机启动
systemctl daemon-reload && systemctl enable --now etcd

# 启动有问题,使用 journalctl -u 服务名排查
journalctl -u etcd

# 查看etcd集群的服务状态
systemctl status etcd

# 查看相应的端口号是否启动
netstat -auntlp | grep 2379

# 查看etcd集群状态
etcdctl --endpoints="10.0.0.51:2379,10.0.0.52:2379,10.0.0.53:2379" \
--cacert=/etc/rancher/pki/etcd/ca.pem \
--cert=/etc/rancher/pki/etcd/etcd.pem \
--key=/etc/rancher/pki/etcd/etcd-key.pem \
endpoint status --write-out=table
```

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679669671836-1d94b1e4-0aab-4712-b816-ff7538e5b28b.png)

### 1.4.2 💎K3s 集群使用外部etcd高可用集群进行数据存储

参考资料：https://docs.rancher.cn/docs/k3s/installation/datastore/_index

如果你想使用外部数据存储，如 PostgreSQL、MySQL 或 etcd，你必须设置`datastore-endpoint`参数，以便 K3s 知道如何连接到它。你也可以指定参数来配置连接的认证和加密。下表总结了这些参数，它们可以作为 CLI 标志或环境变量传递。

| **CLI Flag**           | **环境变量**           | **描述**                                                     |
| ---------------------- | ---------------------- | ------------------------------------------------------------ |
| `--datastore-endpoint` | K3S_DATASTORE_ENDPOINT | 指定一个 PostgreSQL、MySQL 或 etcd 连接字符串。用于描述与数据存储的连接。这个字符串的结构是特定于每个后端的，详情如下。 |
| `--datastore-cafile`   | K3S_DATASTORE_CAFILE   | TLS 证书颁发机构（CA）文件，用于帮助确保与数据存储的通信安全。如果你的数据存储通过 TLS 服务请求，使用由自定义证书颁发机构签署的证书，你可以使用这个参数指定该 CA，这样 K3s 客户端就可以正确验证证书。 |
| `--datastore-certfile` | K3S_DATASTORE_CERTFILE | TLS 证书文件，用于对数据存储进行基于客户端证书的验证。要使用这个功能，你的数据存储必须被配置为支持基于客户端证书的认证。如果你指定了这个参数，你还必须指定datastore-keyfile参数。 |
| `--datastore-keyfile`  | K3S_DATASTORE_KEYFILE  | TLS 密钥文件，用于对数据存储进行基于客户端证书的认证。更多细节请参见前面的datastore-certfile参数。 |

作为最佳实践，我们建议将这些参数设置为环境变量，而不是命令行参数，这样你的数据库证书或其他敏感信息就不会作为进程信息的一部分暴露出来。

- 部署Docker容器运行时环境

```shell
# 安装Docker容器运行时环境
# 在 K3s 节点上安装 Docker。可以使用 Rancher 的一个Docker 安装脚本来安装 Docker：
curl https://releases.rancher.com/install-docker/19.03.sh | sh

# 配置镜像加速器
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://po13h3y1.mirror.aliyuncs.com","http://hub-mirror.c.163.com","https://mirror.ccs.tencentyun.com","http://f1361db2.m.daocloud.io"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker && sudo systemctl enable docker
```

- 创建K3s集群环境使用etcd数据库

```shell
# K3s Master
# $ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
#   INSTALL_K3S_VERSION="v1.21.14+k3s1" \
#   INSTALL_K3S_EXEC="server --docker --node-name k3s1" \
#   sh -s - server --write-kubeconfig ~/.kube/config \
#   --datastore-endpoint="https://10.0.0.51:2379,https://10.0.0.52:2379,https://10.0.0.53:2379" \
#   --datastore-cafile='/etc/rancher/pki/etcd/ca.pem' \
#   --datastore-certfile='/etc/rancher/pki/etcd/etcd.pem' \
#   --datastore-keyfile='/etc/rancher/pki/etcd/etcd-key.pem' \
#   --token=rancher 

# $ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
#   INSTALL_K3S_VERSION="v1.21.14+k3s1" \
#   INSTALL_K3S_EXEC="server --docker --node-name k3s1" \
#   K3S_DATASTORE_ENDPOINT="k3s-master01=https://10.0.0.51:2379,k3s-master02=https://10.0.0.52:2379,k3s-master03=https://10.0.0.53:2379" \
#   K3S_DATASTORE_CERTFILE="/etc/rancher/pki/etcd/etcd.pem" \
#   K3S_DATASTORE_KEYFILE="/etc/rancher/pki/etcd/etcd-key.pem'" \
#   K3S_DATASTORE_CAFILE="/etc/rancher/pki/etcd/ca.pem" \
#   sh -s -

# 使用配置文件(推荐方式)
$ mkdir -pv /opt/rancher/k3s/
$ cat <<-'EOF'> /opt/rancher/k3s/config.yaml
# 保留的快照数量
etcd-snapshot-retention: 7
# 以 Cron 表达式的形式配置触发定时快照的时间点
etcd-snapshot-schedule-cron: "*/10 * * * *"
# 使用etcd作为外置数据库
datastore-endpoint: "etcd"
# 初始化集群
cluster-init: true
# 设置Token值
token: rancher
EOF

$ curl -sfL https://get.k3s.io | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" K3S_TOKEN=rancher \
  sh -s - --cluster-init --docker --node-name k3s1 --config=/opt/rancher/k3s/config.yaml

# 查看Token信息
$ cat /var/lib/rancher/k3s/server/token
K103272c8968f1442396aca76d694663cbc56933c0bfa38c783cf9d7c6b421e7e8a::server:rancher

# K3s Master
$ curl -sfL https://get.k3s.io | \
  INSTALL_K3S_MIRROR=cn K3S_TOKEN=rancher \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  INSTALL_K3S_EXEC="--node-ip=10.0.0.52 --docker --node-name k3s2" \
  sh -s - --server https://10.0.0.51:6443

$ curl -sfL https://get.k3s.io | \
  INSTALL_K3S_MIRROR=cn K3S_TOKEN=rancher \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  INSTALL_K3S_EXEC="--node-ip=10.0.0.53 --docker --node-name k3s3" \
  sh -s - --server https://10.0.0.51:6443
# 查看Node节点信息
$ kubectl get nodes -o wide
NAME   STATUS   ROLES                       AGE     VERSION         INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
k3s1   Ready    control-plane,etcd,master   6m2s    v1.21.14+k3s1   10.0.0.51     <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://19.3.15
k3s2   Ready    control-plane,etcd,master   22s     v1.21.14+k3s1   10.0.0.52     <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://19.3.15
k3s3   Ready    control-plane,etcd,master   2m18s   v1.21.14+k3s1   10.0.0.53     <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://19.3.15

# 创建测试Pod
$ kubectl create deployment nginx-deployment --image=nginx --replicas=5
$ kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-84cd76b964-647kg   1/1     Running   0          10m
nginx-deployment-84cd76b964-8f4jl   1/1     Running   0          10m
nginx-deployment-84cd76b964-9f4qs   1/1     Running   0          10m
nginx-deployment-84cd76b964-lqwpf   1/1     Running   0          10m
nginx-deployment-84cd76b964-pntjd   1/1     Running   0          10m

# 过一段时间后查看快照文件情况
$ ls -lt /var/lib/rancher/k3s/server/db/snapshots
total 6256
-rw------- 1 root root 3956768 Mar 25 14:30 etcd-snapshot-k3s1-1679725800
-rw------- 1 root root 2441248 Mar 25 14:20 etcd-snapshot-k3s1-1679725200
```

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679726243559-04f648d3-45af-413d-a1a9-79c4f64b84ba.png)

**提示：如上需要在所有master节点执行，/var/lib/rancher/k3s/server/manifests/为k3s中的静态pud路径。**

`--write-kubeconfig ~/.kube/config`效果为将配置文件写到k8s默认会用的位置，而不是k3s默认的位置`/etc/rancher/k3s/k3s.yaml`。后者会导致istio、helm需要额外设置或无法运行。

释义：脚本安装可配置参数：

- `INSTALL_K3S_MIRROR`：中国区用户设置`INSTALL_K3S_MIRROR=cn`可加速K3S二进制文件下载，也可以直接手动下载二进制文件。
- `INSTALL_K3S_SKIP_DOWNLOAD`：如果设置为`true`，则不会下载`K3s hash`文件或K3S二进制文件。
- `INSTALL_K3S_SYMLINK`：如果设置为`skip`将不会创建软链接，`force`将强制覆盖，如果`path`中不存在命令，则默认值为符号链接。
- `INSTALL_K3S_SKIP_START`：如果设置为`true`，将不会自动启动`K3s`服务。
- `INSTALL_K3S_VERSION`：可从github下载的K3s版本。如果未指定，将尝试下载`latest`版本。
- `INSTALL_K3S_BIN_DIR`：将K3S的二进制文件，软链接和卸载脚本安装到的目录，`/usr/local/bin`用作默认目录。
- `INSTALL_K3S_BIN_DIR_READ_ONLY`：如果设置为true，则不会写入文件到`INSTALL_K3S_BIN_DIR`，如果需要强制写入则设置`INSTALL_K3S_SKIP_DOWNLOAD=true`。
- `INSTALL_K3S_SYSTEMD_DIR`：将systemd服务和环境变量文件安装到的目录，`/etc/systemd/system`用作默认目录。
- `INSTALL_K3S_EXEC`：当未指定`INSTALL_K3S_EXEC`变量，或设置了`K3S_URL`变量，或在`INSTALL_K3S_EXEC`变量中没有添加server执行命令，那么k3s默认将以agent角色运行。反之，会默认以server角色运行。最终，`systemd`命令会被解析为EXEC命令和脚本参数的组合（$@）。
- `INSTALL_K3S_NAME`：如果未指定，将默认使用`K3s exec`命令创建的`systemd`服务的名称。如果指定，名称将以`k3s-为前缀`。
- `INSTALL_K3S_TYPE`：要创建的`systemd`服务类型，如果未指定，将默认使用`K3s exec`命令。

**提示：默认K3S将以**`**flannel**`**作为CNI运行，并使用**`**VXLAN**`**作为默认后端。本实验采用**`**外部etcd数据库作为存储**`**，更多数据存储类型可参考：**`**https://docs.rancher.cn/k3s/installation/datastore.html**`**。**

- 创建快照

K3s 默认启用快照。快照目录默认为 `/var/lib/rancher/k3s/server/db/snapshots`。要配置快照间隔或保留的快照数量，请参考：

| **参数**                        | **描述**                                                     |
| ------------------------------- | ------------------------------------------------------------ |
| `--etcd-disable-snapshots`      | 禁用自动 etcd 快照                                           |
| `--etcd-snapshot-schedule-cron` | 以 Cron 表达式的形式配置触发定时快照的时间点，例如：每 5 小时触发一次`* */5 * * *`，默认值为每 12 小时触发一次：`0 */12 * * *` |
| `--etcd-snapshot-retention`     | 保留的快照数量，默认值为 5。                                 |
| `--etcd-snapshot-dir`           | 保存数据库快照的目录路径。(默认位置：`${data-dir}/db/snapshots`) |
| `--cluster-reset`               | 忘记所有的对等体，成为新集群的唯一成员，也可以通过环境变量`[$K3S_CLUSTER_RESET]`进行设置。 |
| `--cluster-reset-restore-path`  | 要恢复的快照文件的路径                                       |

- 从快照恢复集群

当 K3s 从备份中恢复时，旧的数据目录将被移动到`/var/lib/rancher/k3s/server/db/etcd-old/`。然后 K3s 会尝试通过创建一个新的数据目录来恢复快照，然后从一个带有一个 etcd 成员的新 K3s 集群启动 etcd。

要从备份中恢复集群，运行 K3s 时，请使用`--cluster-reset`选项运行 K3s，同时给出`--cluster-reset-restore-path`，如下：

```shell
$ k3s server \
  --cluster-reset \
  --cluster-reset-restore-path=<PATH-TO-SNAPSHOT>

$ ls -lt /var/lib/rancher/k3s/server/db/snapshots
total 6256
-rw------- 1 root root 3956768 Mar 25 14:30 etcd-snapshot-k3s1-1679725800
-rw------- 1 root root 2441248 Mar 25 14:20 etcd-snapshot-k3s1-1679725200

# 开始进行测试
$ kubectl delete deployment nginx-deployment
$ kubectl get pod
No resources found in default namespace.

# 需要对所有的Node节点执行(停止k3s服务)
$ systemctl stop k3s.service

# 恢复集群
$ k3s server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/k3s/server/db/snapshots/etcd-snapshot-k3s1-1679725800
[......]
INFO[2023-03-27T17:02:20.754099984+08:00] Reconciling bootstrap data between datastore and disk
INFO[2023-03-27T17:02:20.770556535+08:00] Cluster reset: backing up certificates directory to /var/lib/rancher/k3s/server/tls-1679907740
INFO[2023-03-27T17:02:20.781276897+08:00] Etcd is running, restart without --cluster-reset flag now. Backup and delete ${datadir}/server/db on each peer etcd server and rejoin the nodes
$ systemctl start k3s.service

# 需要对k3s-server-2 | k3s-server-3执行
$ rm -rf /var/lib/rancher/k3s/server/db/
$ systemctl start k3s.service

# 查看Node节点的情况
$ kubectl get nodes
NAME   STATUS   ROLES                       AGE     VERSION
k3s1   Ready    control-plane,etcd,master   14m     v1.21.14+k3s1
k3s2   Ready    control-plane,etcd,master   9m13s   v1.21.14+k3s1
k3s3   Ready    control-plane,etcd,master   11m     v1.21.14+k3s1
# 查看Pod的状态
$ kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP           NODE   NOMINATED NODE   READINESS GATES
nginx-deployment-84cd76b964-5hmtc   1/1     Running   0          8m53s   10.42.2.5    k3s2   <none>           <none>
nginx-deployment-84cd76b964-j9pzb   1/1     Running   0          8m53s   10.42.1.5    k3s3   <none>           <none>
nginx-deployment-84cd76b964-k7zzk   1/1     Running   0          8m53s   10.42.2.6    k3s2   <none>           <none>
nginx-deployment-84cd76b964-krfbw   1/1     Running   0          8m53s   10.42.1.6    k3s3   <none>           <none>
nginx-deployment-84cd76b964-wljvf   1/1     Running   0          8m53s   10.42.0.10   k3s1   <none>           <none>
```

**结果:** 日志中出现一条信息，**Etcd 正在运行，现在需要在没有** `**--cluster-reset**` **标志的情况下重新启动。备份和删除每个对等 etcd 服务器上的** `**${datadir}/server/db**` **并重新加入节点**

### 1.4.3 设置kubectl自动补全

```shell
# 安装自动补全工具
yum install bash-completion

# Kubectl自动补全
echo 'source <(kubectl completion bash)' >>~/.bashrc
kubectl completion bash >/etc/bash_completion.d/kubectl
source /usr/share/bash-completion/bash_completion
```

## 1.5 S3 兼容 API 支持

K3s 支持向具有 S3 兼容 API 的系统写入 etcd 快照和从系统中恢复 etcd 快照。S3 支持按需和计划快照。

下面的参数已经被添加到 server 子命令中。这些标志也存在于 `etcd-snapshot` 子命令中，但是 `--etcd-s3` 部分被删除以避免冗余。

```shell
# 需要支持S3的支持，就需要设置https协议，不支持http协议
$ k3s etcd-snapshot \
  --s3 \
  # --s3-endpoint minio.kingsd.top:9000 \
  --s3-bucket=<S3-BUCKET-NAME> \
  --s3-access-key=<S3-ACCESS-KEY> \	
  --s3-secret-key=<S3-SECRET-KEY>
```

启动 Minio 对象存储

```shell
$ docker run -p 9000:9000 -p 9090:9090 \
 --net=host \
 --name minio \
 -d --restart=always \
 -e "MINIO_ACCESS_KEY=minioadmin" \
 -e "MINIO_SECRET_KEY=minioadmin" \
 -v /home/minio/data:/data \
 -v /home/minio/config:/root/.minio \
 minio/minio server \
 /data --console-address ":9090" -address ":9000"

$ docker ps -l
CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS         PORTS     NAMES
8c98e1abcda4   minio/minio   "/usr/bin/docker-ent…"   6 seconds ago   Up 5 seconds             minio
# 访问：http://10.0.0.51:9090/login 用户名：密码  minioadmin：minioadmin
```

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679908281863-9d0fb800-fc10-418e-8d8a-0a8c7da1060a.png)

```shell
$ k3s etcd-snapshot \
  --s3 \
  --s3-endpoint 10.0.0.51:9090 \
  --s3-bucket=k3s-backup \
  --s3-access-key=minioadmin \
  --s3-secret-key=minioadmin
```

要从 S3 中执行按需的 etcd 快照还原，首先确保 K3s 没有运行。然后运行以下命令：

```shell
$ k3s server \
  --cluster-init \
  --cluster-reset \
  --etcd-s3 \
  # --etcd-s3-endpoint minio.kingsd.top:9000 \
  --cluster-reset-restore-path=<SNAPSHOT-NAME> \
  --etcd-s3-bucket=<S3-BUCKET-NAME> \
  --etcd-s3-access-key=<S3-ACCESS-KEY> \
  --etcd-s3-secret-key=<S3-SECRET-KEY>
```

## 1.6 Etcd 快照和恢复子命令[#](https://docs.rancher.cn/docs/k3s/backup-restore/_index#etcd-快照和恢复子命令)

k3s 支持一组用于处理 etcd 快照的子命令

| **子命令**  | **描述**                     |
| ----------- | ---------------------------- |
| delete      | 删除指定的快照               |
| ls, list, l | 列出快照                     |
| prune       | 删除超过配置的保留次数的快照 |
| save        | 触发一个即时的 etcd 快照     |

*💡注* save子命令与k3s etcd-snapshot相同。后者最终将被弃用，取而代之的是前者。

无论 etcd 快照是存储在本地还是存储在 S3 兼容的对象存储中，这些命令都将按预期执行。

关于 etcd 快照子命令的其他信息，请运行`k3s etcd-snapshot`。

从 S3 中删除一个快照。

```shell
k3s etcd-snapshot delete \
 --s3 \
 --s3-bucket=<S3-BUCKET-NAME>\
 --s3-access-key=<S3-ACCESS-KEY>\
 --s3-secret-key=<S3-SECRET-KEY>\
<SNAPSHOT-NAME>
```

用默认的保留策略（5）prune 本地快照。prune子命令接受一个额外的标志--snapshot-retention，允许覆盖默认的保留策略。

```shell
k3s etcd-snapshot prune
k3s etcd-snapshot prune --snapshot-retention 10
```