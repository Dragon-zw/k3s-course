Video Reference：[11-安装-仪表盘及卸载K3s](https://www.bilibili.com/video/BV1SL4y1b7g7/?spm_id_from=333.788&vd_source=9560c118fae1db9638f05a6ba2527085)

GitHub README.md：[11-安装-仪表盘及卸载K3s](https://github.com/kingsd041/k3s-tutorial/tree/main/11-安装-仪表盘及卸载K3s)

# 0 仪表盘及卸载 K3S

仪表盘

推荐使用三种仪表盘工具，分别是对应是 `Kubernetes Dashboard`、`kube-explorer` 和 `Rancher UI`，其各自各有优劣。

- Kubernetes Dashboard
- kube-explorer
- Rancher UI

## 0 创建 K3s 集群

范例：部署本地环境K3s集群

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
  INSTALL_K3S_EXEC="server --docker" \
  K3S_TOKEN=rancher sh -s -

# 查看 K3s Master Token 信息
$ cat /var/lib/rancher/k3s/server/token
K1046b15950a113b87d32b8d245b15490b76722b81f16b79f0e1eb912d2fe0d6b9a::server:rancher

# K3s Worker
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  K3S_URL=https://10.0.0.51:6443 \
  K3S_TOKEN=rancher INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  INSTALL_K3S_EXEC="--node-ip=10.0.0.52 --docker" \
  sh -

$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  K3S_URL=https://10.0.0.51:6443 \
  K3S_TOKEN=rancher INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  INSTALL_K3S_EXEC="--node-ip=10.0.0.53 --docker" \
  sh -
```

范例：本地环境情况

```shell
$ kubectl get nodes
NAME   STATUS   ROLES                  AGE   VERSION
k3s1   Ready    control-plane,master   14h   v1.21.14+k3s1
k3s2   Ready    <none>                 14h   v1.21.14+k3s1
k3s3   Ready    <none>                 14h   v1.21.14+k3s1
```

## 1. Kubernetes Dashboard

Dashboard 是基于网页的Kubernetes 用户界面。你可以使用Dashboard 将容器应用部署到Kubernetes集群中，也可以对容器应用排错，还能管理集群资源。你可以使用Dashboard 获取运行在集群中的应用的概览信息，也可以创建或者修改Kubernetes资源(如 Deployment，Job，DaemonSet 等等)。例如，你可以对Deployment实现弹性伸缩、发起滚动升级、重启Pod或者使用向导创建新的应用。

Dashboard同时展示了Kubernetes 集群中的资源状态信息和所有报错信息。

参考资料：

http://docs.rancher.cn/docs/k3s/installation/kube-dashboard/_index/

https://kubernetes.io/zh-cn/docs/tasks/access-application-cluster/web-ui-dashboard/

```shell
# 部署Kubernetes仪表盘
GITHUB_URL=https://github.com/kubernetes/dashboard/releases
VERSION_KUBE_DASHBOARD=$(curl -w '%{url_effective}' -I -L -s -S ${GITHUB_URL}/latest -o /dev/null | sed -e 's|.*/||')
sudo k3s kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/${VERSION_KUBE_DASHBOARD}/aio/deploy/recommended.yaml
```

### 1.1 仪表盘 RBAC 配置

**重要：** 本指南中创建的 admin-user 将在仪表盘中拥有管理权限。

创建以下资源清单文件：

```
dashboard.admin-user.yml
# dashboard.admin-user.yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
dashboard.admin-user-role.yml
# dashboard.admin-user-role.yml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
```

部署 `admin-user` 配置：

```yaml
# 仪表盘RBAC配置
# 本指南中创建的admin-user将在仪表盘中拥有管理权限
sudo k3s kubectl create -f dashboard.admin-user.yml -f dashboard.admin-user-role.yml
```

### 1.2 获得 Bearer Toke

*v1.24+*

```shell
# 获得Bearer-Token
sudo k3s kubectl -n kubernetes-dashboard create token admin-user
```

*v1.23 及之前的版本*

```shell
# 获得Bearer-Token
sudo k3s kubectl -n kubernetes-dashboard describe secret admin-user-token |grep '^token'
```

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679649587983-cc6c47f8-2ab6-4598-8912-f9bc05bc4e28.png)

### 1.3 高级：远程访问仪表盘[#](http://docs.rancher.cn/docs/k3s/installation/kube-dashboard/_index/#高级：远程访问仪表盘)

请参阅仪表盘文档。使用[端口转发](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)来访问集群中的应用程序。[ 也可以通过将`ClusterIP`设置为`NodePort`的方式进行暴露 ]

```shell
# 端口转发的方式暴露
kubectl port-forward -n kubernetes-dashboard --address 0.0.0.0 svc/kubernetes-dashboard 8443:443

# 以NodePort的方式暴露
# kubectl edit svc -n kubernetes-dashboard kubernetes-dashboard
# type: ClusterIP -> type: NodePort

$ kubectl patch svc kubernetes-dashboard --namespace kubernetes-dashboard -p '{"spec":{"type": "NodePort"}}'
$ kubectl get svc --namespace kubernetes-dashboard kubernetes-dashboard
NAME                   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   NodePort   10.43.213.160   <none>        443:30891/TCP   2m12s
```

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679650266335-2420d1e1-8c9e-4f89-b5e1-e596a83a0249.png)

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679650482888-ccc4b833-0cc7-4533-96e8-df3cbdf9b011.png)

### 1.4 升级 Kubernetes Dashboard 

```shell
# 升级仪表盘
$ sudo k3s kubectl delete ns kubernetes-dashboard
$ GITHUB_URL=https://github.com/kubernetes/dashboard/releases
$ VERSION_KUBE_DASHBOARD=$(curl -w '%{url_effective}' -I -L -s -S ${GITHUB_URL}/latest -o /dev/null | sed -e 's|.*/||')
$ sudo k3s kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/${VERSION_KUBE_DASHBOARD}/aio/deploy/recommended.yaml -f dashboard.admin-user.yml -f dashboard.admin-user-role.yml
```

### 1.5 删除仪表盘和admin-user配置

```shell
# 删除仪表盘和admin-user配置
$ sudo k3s kubectl delete ns kubernetes-dashboard
$ sudo k3s kubectl delete clusterrolebinding kubernetes-dashboard
$ sudo k3s kubectl delete clusterrole kubernetes-dashboard
```

## 2. kube-explorer

### 2.1 kube-explorer 的介绍

项目地址：https://github.com/cnrancher/kube-explorer

`kube-explorer` 是 `Kubernetes` 的便携式资源管理器，没有任何依赖。 并提供了一个几乎完全无状态的 `Kubernetes` 资源管理器。

### 2.2 kube-explorer 使用

从 [发布页面](https://github.com/cnrancher/kube-explorer/releases) 下载二进制文件

```shell
# 从发布页面下载二进制文件
$ wget https://github.com/cnrancher/kube-explorer/releases/download/v0.3.1/kube-explorer-linux-amd64
$ chmod +x kube-explorer-linux-amd64 && mv kube-explorer-linux-amd64 kube-explorer

# cp -av ./kube-explorer /usr/local/bin/kube-explorer
```

运行：

```shell
# --kubeconfig 可以不配置(自己可以找到)
$ ./kube-explorer --kubeconfig=/etc/rancher/k3s/k3s.yaml --http-listen-port=9898 --https-listen-port=0

# 后台运行
$ nohup ./kube-explorer --kubeconfig=/etc/rancher/k3s/k3s.yaml --http-listen-port=9898 --https-listen-port=0 &
```

然后，打开浏览器访问 `http://[Node_IPAddress]:9898`

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679650800154-bef1f173-af1d-47bd-8cfb-3b9ef69b3f27.png)

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679650878950-067ee089-91d0-4c52-81c6-07748c1862f0.png)

## 3. Rancher UI

### 3.1 Rancher UI简介

可以将 K3s 导入到 Rancher UI 中去管理，参考 [Rancher 官网](http://docs.rancher.cn/docs/rancher2/cluster-provisioning/imported-clusters/_index/#导入-k3s-集群)

导入 K3s 集群时，Rancher 会将其识别为 K3s，除了其他导入的集群支持的功能之外，Rancher UI 还提供以下功能：

- 能够升级 K3s 版本。
- 能够配置在升级集群时，同时可以升级的最大节点数。
- 在主机详情页，能够查看（不能编辑）启动 K3s 集群时每个节点的 K3s 配置参数和环境变量。

### 3.2 启动 Rancher UI

参考资料：

https://docs.rancher.cn/docs/rancher2/best-practices/use-in-china/_index

如果你在不考虑身份验证的开发或测试环境中安装 Rancher，可以使用 Rancher 生成的自签名证书安装 Rancher。这种安装方式避免了自己生成证书的麻烦。

登录到你的主机，然后运行以下命令：

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679652910072-f04cfbe9-b02a-4755-9113-684e00f530bd.png)

对于 Helm CLI 安装，需要在运行 Rancher Server 的任何节点上安装 Docker。

有两种安装 Docker 的选择。一种选择是参考[Docker 官方文档](https://docs.docker.com/install/)来了解如何在 Linux 上安装 Docker。这些安装步骤将根据 Linux 发行版而有所不同。

另一种选择是使用 Rancher 提供的 Docker 安装脚本，该脚本可用于安装较新的 Docker 版本。

例如，此命令可用于在 Ubuntu 上安装 Docker 20.10：

```shell
curl https://releases.rancher.com/install-docker/20.10.sh | sh
```

Rancher 为 Kubernetes 支持的每个上游 Docker 版本都有安装脚本。要想知道是否有安装某个 Docker 版本的脚本，可以参考这个[GitHub 仓库](https://github.com/rancher/install-docker)，其中包含了 Rancher 所有的 Docker 安装脚本。

```shell
# 官方文档启动Rancher(需要对rancher进行数据持久化)
# 启动rancher容器

# 需要使用新的机器，不能使用K3s集群内的机器作为Rancher容器的部署
# 原因会使得Rancher容器内部的Pod创建失败
# Rancher v2.7.1
$ docker run -d --restart=always \
  -p 80:80 -p 443:443 \
  -e CATTLE_SYSTEM_DEFAULT_REGISTRY=registry.cn-hangzhou.aliyuncs.com \
  -e CATTLE_BOOTSTRAP_PASSWORD=admin \
  -v /rancher/data:/var/lib/rancher/ \
  --name rancher \
  --privileged=true \
  rancher/rancher:v2.7.1-linux-amd64 # v2.7.1

# Rancher v2.5.2
# $ mkdir -pv /rancher/rancher-v2.5.2
# $ docker run --privileged -d \
#   --restart=unless-stopped \
#   -p 80:80 -p 443:443 -v /rancher/rancher-v2.5.2:/var/lib/rancher \
#   --name rancher rancher/rancher:v2.5.2-linux-amd64

$ docker ps -l
CONTAINER ID        IMAGE                                COMMAND             CREATED             STATUS              PORTS                                      NAMES
0c8e4cb6a99e        rancher/rancher:v2.7.1-linux-amd64   "entrypoint.sh"     18 seconds ago      Up 30 seconds       0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp   rancher

# 进入到容器查看相应的情况
$ docker exec -it rancher bash
0c8e4cb6a99e:/var/lib/rancher # kubectl get nodes
NAME         STATUS   ROLES                       AGE   VERSION
local-node   Ready    control-plane,etcd,master   57s   v1.24.4+k3s1
0c8e4cb6a99e:/var/lib/rancher # kubectl get pod -A
NAMESPACE                   NAME                                READY   STATUS      RESTARTS   AGE
cattle-fleet-local-system   fleet-agent-77c59d5455-c6t95        1/1     Running     0          33s
cattle-fleet-system         fleet-controller-5df4897959-hgj54   1/1     Running     0          102s
cattle-fleet-system         gitjob-5df6df5996-vzhdf             1/1     Running     0          102s
cattle-system               helm-operation-49mdl                0/2     Completed   0          64s
cattle-system               helm-operation-cwp2f                0/2     Completed   0          72s
cattle-system               helm-operation-pmgdx                0/2     Completed   0          2m6s
cattle-system               helm-operation-xclzq                0/2     Completed   0          58s
cattle-system               rancher-webhook-6987d476d9-gbjr2    1/1     Running     0          56s
kube-system                 coredns-b96499967-lxntk             1/1     Running     0          2m24s

# 查看容器的日志
$ docker logs -f rancher
```

浏览器访问 `http://[NODE_IP_ADDRESS]` 的Rancher WebUI

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679947226271-3326e598-94f3-4ddc-bb34-cc1e0b63a3fe.png)

### 3.3 通过 Rancher WebUI 添加 Kuberrnetes 集群

需要查看支持的矩阵：

https://rancher.com/support-maintenance-terms/all-supported-versions/rancher-v2.5.7/

点击"`添加集群`"→在使用现有的 Kubernetes 集群中选择 "`导入`"→"集群名称自定义即可"(k3s-cluster)

根据 Rancher UI 生成的命令进行配置即可

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679947373972-a949a644-04d1-475b-b169-e131e056fa86.png)

```shell
### 需要在 Master 节点执行
# 在现有的受支持的 Kubernetes 集群上运行下面的 kubectl 命令，将其导入 Rancher:
$ kubectl apply -f https://10.0.0.53/v3/import/jxcxtwpfl8wbtl9pwsqcd598v8xf7h8w64fm6grcrgc9r2cqvxjrp5.yaml

# 目前环境使用的不受信任/自签名的 SSL 证书
# 如果由于您的 Rancher 安装使用不受信任/自签名的 SSL 证书而出现 由未知权限签名的证书 错误，请运行下面的命令以绕过证书检查:
$ curl --insecure -sfL https://10.0.0.53/v3/import/jxcxtwpfl8wbtl9pwsqcd598v8xf7h8w64fm6grcrgc9r2cqvxjrp5.yaml | kubectl apply -f -
```

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679947560653-1c1a7620-1112-49d7-b368-7d7c2a3a72aa.png)

```shell
# 在Master节点查看所有的cattle的运行状态
$ kubectl get all -n cattle-system
```

浏览器访问 `http://[NODE_IP_ADDRESS]` 的Rancher WebUI

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679948457619-382c49af-7568-4f64-a4ae-32983c6c4973.png)

可以通过 Rancher WebUI 查看 Node 节点的信息

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679950077047-83136a7e-1745-4525-a741-69ef1e3417f6.png)

可以通过 Rancher WebUI 升级相应的集群节点的版本

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679948570620-e64f3c5a-9972-47c5-a8a8-518112dc14f0.png)

- Rancher WebUI 在升级集群版本时，会比 GitHub Release 的版本要低，这是因为 Rancher Server 会定期的查询数据源的 K3s 版本号

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679949611114-eae77e8f-19d2-4517-8765-0ca83011a8e8.png)

在 https://releases.rancher.com/kontainer-driver-metadata/release-v2.5/data.json 数据源中查看到 K3s 版本的信息

更改的 URL 来实现对 K3s 版本的信息

Github Reference：https://github.com/rancher/kontainer-driver-metadata

例如可以使用的URL：https://raw.githubusercontent.com/rancher/kontainer-driver-metadata/dev-v2.7/data/data.json

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679949841010-708ab3ea-0d5f-40b7-bf3b-2171c158c3ef.png)

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679949860672-eeb36e9f-96c2-4388-851c-e34f8f08c8cb.png)

💎可以参考`12-K3S 升级k3s集群`章节查看 K3s 集群Node节点的版本升级和降级操作。

```shell
# 导入K3s集群时，Rancher会将其识别为K3s类型，并且附件额外功能
# 1.能够升级K3s版本
# 2.可配置升级集群时升级的最大节点数
# 3.在主机详情页能够查看启动K3s集群时每个节点的配置参数和环境变量

# 配置K3s集群以允许导入到Rancher
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn sh -s - \
  --write-kubeconfig-mode 644
```

## 4. 小总结

- 大型 K3s环境的话，可以使用 Rancher 进行监控平台
- 中小型 K3s环境的话，可以使用 kube-explorer 以及使用 Kubernetes Dashboard 进行监控平台

# 1 卸载 K3S

要从 server 节点卸载 K3s，请运行：

```shell
# 主节点
/usr/local/bin/k3s-uninstall.sh
```

要从 agent 节点卸载 K3s，请运行：

```shell
# 工作节点
/usr/local/bin/k3s-agent-uninstall.sh
```

## 1.1 清除 K3s 集群环境脚本(Docker)

官网文档：[清理节点](http://docs.rancher.cn/docs/rancher2.5/cluster-admin/cleaning-cluster-nodes/_index/#清理脚本)

- 需要管理员注意 K3s 集群的 Docker 创建的容器，要管理员手动判断 Docker 容器的生命周期。

```shell
# 包括docker等信息一并清理

#!/bin/bash

KUBE_SVC='
kubelet
kube-scheduler
kube-proxy
kube-controller-manager
kube-apiserver
'

for kube_svc in ${KUBE_SVC};
do
  # 停止服务
  if [[ `systemctl is-active ${kube_svc}` == 'active' ]]; then
    systemctl stop ${kube_svc}
  fi
  # 禁止服务开机启动
  if [[ `systemctl is-enabled ${kube_svc}` == 'enabled' ]]; then
    systemctl disable ${kube_svc}
  fi
done

# 停止所有容器
docker stop $(docker ps -aq)

# 删除所有容器
docker rm -f $(docker ps -qa)

# 删除所有容器卷
docker volume rm $(docker volume ls -q)

# 卸载mount目录
for mount in $(mount | grep tmpfs | grep '/var/lib/kubelet' | awk '{ print $3 }') /var/lib/kubelet /var/lib/rancher;
do
  umount $mount;
done

# 备份目录
mv /etc/kubernetes /etc/kubernetes-bak-$(date +"%Y%m%d%H%M")
mv /var/lib/etcd /var/lib/etcd-bak-$(date +"%Y%m%d%H%M")
mv /var/lib/rancher /var/lib/rancher-bak-$(date +"%Y%m%d%H%M")
mv /opt/rke /opt/rke-bak-$(date +"%Y%m%d%H%M")

# 删除残留路径
rm -rf /etc/ceph \
    /etc/cni \
    /opt/cni \
    /run/secrets/kubernetes.io \
    /run/calico \
    /run/flannel \
    /var/lib/calico \
    /var/lib/cni \
    /var/lib/kubelet \
    /var/log/containers \
    /var/log/kube-audit \
    /var/log/pods \
    /var/run/calico \
    /usr/libexec/kubernetes

# 清理网络接口
no_del_net_inter='
lo
docker0
eth
ens
bond
'

network_interface=`ls /sys/class/net`

for net_inter in $network_interface;
do
  if ! echo "${no_del_net_inter}" | grep -qE ${net_inter:0:3}; then
    ip link delete $net_inter
  fi
done

# 清理残留进程
port_list='
80
443
6443
2376
2379
2380
8472
9099
10250
10254
'

for port in $port_list;
do
  pid=`netstat -atlnup | grep $port | awk '{print $7}' | awk -F '/' '{print $1}' | grep -v - | sort -rnk2 | uniq`
  if [[ -n $pid ]]; then
    kill -9 $pid
  fi
done

kube_pid=`ps -ef | grep -v grep | grep kube | awk '{print $2}'`

if [[ -n $kube_pid ]]; then
  kill -9 $kube_pid
fi

# 清理Iptables表
## 注意：如果节点Iptables有特殊配置，以下命令请谨慎操作
sudo iptables --flush
sudo iptables --flush --table nat
sudo iptables --flush --table filter
sudo iptables --table nat --delete-chain
sudo iptables --table filter --delete-chain
systemctl restart docker
```

## 1.2 使用 docker 作为默认容器运行时

参考：Rancher [官网](http://docs.rancher.cn/docs/rancher2.5/cluster-admin/cleaning-cluster-nodes/_index/#清理脚本)

| **Flag**                           | **默认值**                         | **描述**                                           |
| ---------------------------------- | ---------------------------------- | -------------------------------------------------- |
| --docker                           | N/A                                | 用 docker 代替 containerd                          |
| --container-runtime-endpoint value | N/A                                | 禁用嵌入式 containerd，使用替代的 CRI 实现。       |
| --pause-image value                | "docker.io/rancher/pause:3.1"      | 针对 containerd 或 Docker 的自定义 pause 镜像      |
| --snapshotter value                | N/A                                | 覆盖默认的 containerd 快照程序 (默认: "overlayfs") |
| --private-registry value           | "/etc/rancher/k3s/registries.yaml" | 私有镜像仓库配置文件                               |

在 K3s 节点上安装 Docker。可以使用 Rancher 的一个[Docker 安装脚本](https://github.com/rancher/install-docker)来安装 Docker：

```shell
curl https://releases.rancher.com/install-docker/19.03.sh | sh
```

国内用户，可以使用以下方法加速安装：

```shell
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC="--docker" \
  sh -
[INFO]  Finding release for channel stable
[INFO]  Using v1.25.6+k3s1 as release
[INFO]  Downloading hash rancher-mirror.rancher.cn/k3s/v1.25.6-k3s1/sha256sum-amd64.txt
[INFO]  Skipping binary downloaded, installed k3s matches hash
[INFO]  Skipping /usr/local/bin/kubectl symlink to k3s, already exists
[INFO]  Skipping /usr/local/bin/crictl symlink to k3s, already exists
[INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink from /etc/systemd/system/multi-user.target.wants/k3s.service to /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s

# 查看Docker的容器进程
$ docker ps

# 查看相应的service文件
$ cat /etc/systemd/system/k3s.service
[Unit]
Description=Lightweight Kubernetes
Documentation=https://k3s.io
Wants=network-online.target
After=network-online.target

[Install]
WantedBy=multi-user.target

[Service]
Type=notify
EnvironmentFile=-/etc/default/%N
EnvironmentFile=-/etc/sysconfig/%N
EnvironmentFile=-/etc/systemd/system/k3s.service.env
KillMode=process
Delegate=yes
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
TimeoutStartSec=0
Restart=always
RestartSec=5s
ExecStartPre=/bin/sh -xc '! /usr/bin/systemctl is-enabled --quiet nm-cloud-setup.service'
ExecStartPre=-/sbin/modprobe br_netfilter
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/k3s \
    server \
        '--docker' \
```