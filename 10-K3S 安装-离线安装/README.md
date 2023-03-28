Video Reference：[10-安装-离线安装](https://www.bilibili.com/video/BV1Q44y157LM/?spm_id_from=333.788&vd_source=9560c118fae1db9638f05a6ba2527085)

GitHub README.md：[10-安装-离线安装](https://github.com/kingsd041/k3s-tutorial/tree/main/10-安装-离线安装)

官方文档：https://docs.rancher.cn/docs/k3s/installation/airgap/_index

# 1 K3s 离线安装

## 1.1 概述

你可以使用两种不同的方法在离线环境中安装 K3s。离线环境是不直接连接到 Internet 的任何环境。离线安装的过程主要分为以下两个步骤：

**步骤 1**：部署镜像，本文提供了两种部署方式，分别是**部署私有镜像仓库**和**手动部署镜像**。请在这两种方式中选择一种执行。

- **部署私有镜像仓库**
- **手动部署镜像**

**步骤 2**：安装 K3s，本文提供了两种安装方式，分别是**单节点安装**和**高可用安装**。完成镜像部署后，请在这两种方式中选择一种执行。

- **单节点安装**
- **高可用安装**

**离线升级 K3s 版本**：完成离线安装 K3s 后，您还可以通过脚本升级 K3s 版本，或启用自动升级功能，以保持离线环境中的 K3s 版本与最新的 K3s 版本同步。

## 1.2 通过私有镜像仓库安装 K3s

### 1.2.0 前提条件

本文档假设您已经在离线环境中创建了节点，并且在您的堡垒机上有一个 Docker 私有镜像仓库。

如果你还没有建立私有的 Docker 镜像仓库，请参考[Docker 官方文档](https://docs.docker.com/registry/deploying/#run-an-externally-accessible-registry)建立私有的 Docker 镜像仓库。

### 1.2.1 将所需镜像上传到私有镜像仓库

- K3s 镜像列表可以从 https://github.com/k3s-io/k3s/releases 获取。
- 可以从K3s镜像列表获取到版本，下载上传到私有镜像仓库 

### 1.2.2 创建镜像仓库 YAML

按照[私有镜像仓库配置指南](http://docs.rancher.cn/docs/k3s/installation/private-registry/_index/) 创建并配置`registry.yaml`文件。

```yaml
mkdir -p /etc/rancher/k3s/
cat >> /etc/rancher/k3s/registries.yaml <<EOF
mirrors:
  "docker.io":
    endpoint:
      - "https://harbor.kingsd.top"
configs:
  "docker.io":
    auth:
      username: admin
      password: Harbor@12345
EOF
```

### 1.2.3 安装单节点 K3s

1. 从[K3s GitHub Release](https://github.com/rancher/k3s/releases)页面获取 K3s 二进制文件，K3s 二进制文件需要与离线镜像的版本匹配。
2. 获取 K3s 安装脚本：[https://get.k3s.io。](https://get.k3s.io./)

```shell
# 将脚本内容存放到install.sh的脚本文件中，并设置可执行权限
$ chmod 755 ./install.sh
```

1. 将二进制文件放在每个节点的`/usr/local/bin`中，并确保拥有可执行权限。将安装脚本放在每个节点的任意位置，并将其命名为`install.sh`。
2. 安装 K3s server：

```shell
# 跳过互联网下载k3s二进制文件
INSTALL_K3S_SKIP_DOWNLOAD=true ./install.sh 
```

1. 将 agent 加入到 K3s 集群

```shell
INSTALL_K3S_SKIP_DOWNLOAD=true \
K3S_URL=https://myserver:6443 \
K3S_TOKEN=mynodetoken ./install.sh
```

## 1.3 👑通过手动部署镜像安装 K3s

该离线部署方式更加适合规模比较小的集群方式，可以省略部署镜像仓库的步骤。

请按照以下步骤准备镜像和 K3s 二进制文件：

1. 从 [K3s GitHub Release](https://github.com/rancher/k3s/releases) 页面获取你所运行的 K3s 版本的镜像 tar 文件。
2. 将 tar 文件放在`images`目录下，例如：

```shell
# 该目录的路径不能改变，需要导入镜像的时使用
sudo mkdir -p /var/lib/rancher/k3s/agent/images/
sudo cp ./k3s-airgap-images-$ARCH.tar /var/lib/rancher/k3s/agent/images/

############################################################################################
$ wget https://github.com/k3s-io/k3s/releases/download/v1.26.2%2Bk3s1/k3s-airgap-images-amd64.tar
$ cp -av ./k3s-v1.26.2+k3s1-airgap-images-amd64.tar /var/lib/rancher/k3s/agent/images/
```

1. 将 k3s 二进制文件放在`/usr/local/bin/k3s`路径上，并确保拥有可执行权限。

```shell
# https://github.com/k3s-io/k3s/releases/download/v1.26.2%2Bk3s1/k3s
wget https://github.com/k3s-io/k3s/releases/download/v1.26.2%2Bk3s1/k3s -O /usr/local/bin/k3s
chmod 755 /usr/local/bin/k3s
```

1. 安装 K3s server：

```shell
INSTALL_K3S_SKIP_DOWNLOAD=true ./install.sh 
```

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1680019483777-a91fd077-977f-4a5e-a0a1-73dee4d8fcbf.png)

```shell
# 查看镜像的导入情况
$ crictl images
IMAGE                                        TAG                    IMAGE ID            SIZE
docker.io/rancher/klipper-helm               v0.7.4-build20221121   6f2af12f2834b       252MB
docker.io/rancher/klipper-lb                 v0.4.0                 3449ea2a2bfa7       9.21MB
docker.io/rancher/local-path-provisioner     v0.0.23                9621e18c33880       37.7MB
docker.io/rancher/mirrored-coredns-coredns   1.9.4                  a81c2ec4e946d       49.8MB
docker.io/rancher/mirrored-library-busybox   1.34.1                 827365c7baf13       5.09MB
docker.io/rancher/mirrored-library-traefik   2.9.4                  288889429becf       136MB
docker.io/rancher/mirrored-metrics-server    v0.6.2                 25561daa66605       70.2MB
docker.io/rancher/mirrored-pause             3.6                    6270bb605e12e       686kB

# 查看Token信息
$ cat /var/lib/rancher/k3s/server/token
K1091f73d2eaa358c47315542adf764750ab6e8f4c8f1f06a6020533c4ef1039fde::server:d7ef6dd2922ed77c0fc3b9813c8366b4
```

1. 将 agent 加入到 K3s 集群

```shell
INSTALL_K3S_SKIP_DOWNLOAD=true \
K3S_URL=https://myserver:6443 \
K3S_TOKEN=mynodetoken ./install.sh

#######################################################
INSTALL_K3S_SKIP_DOWNLOAD=true \
K3S_URL=https://10.0.0.51:6443 \
K3S_TOKEN=d7ef6dd2922ed77c0fc3b9813c8366b4 ./install.sh
```

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1680019650522-b9e8d474-2b74-4b5f-81a3-9ae2fb66b366.png)

1. 查看 Node 节点和 Pod 的情况

```shell
# 查看Node的状态
$ kubectl get nodes
NAME   STATUS   ROLES                  AGE     VERSION
k3s1   Ready    control-plane,master   3m53s   v1.26.2+k3s1
k3s2   Ready    <none>                 39s     v1.26.2+k3s1

# 查看Pod的状态
$ kubectl get pod -o wide --all-namespaces
NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE     IP          NODE   NOMINATED NODE   READINESS GATES
kube-system   local-path-provisioner-5d56847996-dppdp   1/1     Running     0          4m25s   10.42.0.4   k3s1   <none>           <none>
kube-system   coredns-5c6b6c5476-fsfqs                  1/1     Running     0          4m25s   10.42.0.6   k3s1   <none>           <none>
kube-system   helm-install-traefik-crd-jmfch            0/1     Completed   0          4m25s   10.42.0.2   k3s1   <none>           <none>
kube-system   svclb-traefik-9de9eebf-h57ms              2/2     Running     0          4m16s   10.42.0.7   k3s1   <none>           <none>
kube-system   traefik-56b8c5fb5c-tzzzg                  1/1     Running     0          4m17s   10.42.0.8   k3s1   <none>           <none>
kube-system   helm-install-traefik-bc6t8                0/1     Completed   1          4m25s   10.42.0.3   k3s1   <none>           <none>
kube-system   metrics-server-7b67f64457-jlphd           1/1     Running     0          4m25s   10.42.0.5   k3s1   <none>           <none>
kube-system   svclb-traefik-9de9eebf-26f9r              2/2     Running     0          76s     10.42.1.2   k3s2   <none>           <none>
```

## 1.4 高可用安装

单节点 k3s server 集群可以满足各种用例，但是对于需要 Kubernetes control-plane 稳定运行的重要环境，您可以在 HA 配置中运行 K3s。一个 K3s HA 集群由以下几个部分组成：

- 两个或多个server 节点，将为 Kubernetes API 提供服务并运行其他 control-plane 服务。
- 零个或多个agent 节点，用于运行您的应用和服务。
- 外部数据存储 (与单个 k3s server 设置中使用的嵌入式 SQLite 数据存储相反，内置的 SQLite 则不会起作用)
- 固定的注册地址，位于 server 节点的前面，以允许 agent 节点向集群注册[ 需要Kubectl连接到一个固定的注册地址 ]
- [这应该是最适合国内用户的 K3s HA 方案](https://mp.weixin.qq.com/s/0Wk2MzfWqMqt8DfUK_2ICA)

Agent 通过固定的注册地址进行注册，但注册后直接与其中一个 server 节点建立连接。这是一个由 k3s agent 进程发起的 websocket 连接，并由作为 agent 进程一部分运行的客户端负载均衡器维护。

指定`INSTALL_K3S_SKIP_DOWNLOAD=true`参数指定使用本地 K3s 二进制文件进行安装。

```shell
INSTALL_K3S_SKIP_DOWNLOAD=true \
INSTALL_K3S_EXEC='server --datastore-endpoint=mysql://username:password@tcp(hostname:3306)/database-name' ./install.sh
```

K3s 还为 kubelets 提供了一个`--resolv-conf`标志，这可能有助于在离线网络中配置 DNS。

## 1.5 升级 K3s

### 1.5.1 通过脚本升级

离线环境的升级可以通过以下步骤完成：

1. 从[K3s GitHub Release](https://github.com/rancher/k3s/releases)页面下载要升级到的 K3s 版本。将 tar 文件放在每个节点的`/var/lib/rancher/k3s/agent/images/`目录下。删除旧的 tar 文件。
2. 复制并替换每个节点上`/usr/local/bin`中的旧 K3s 二进制文件。复制 [https://get.k3s.io](https://get.k3s.io/) 的安装脚本（因为它可能在上次发布后发生了变化）。再次运行脚本。
3. 重启 K3s 服务。

```shell
# 通过脚本升级
# https://github.com/rancher/k3s/releases
# 从Github页面下载要升级到的K3s版本

# 替换
# 复制并替换每个节点上/usr/local/bin中的旧K3s二进制文件
$ mv ./k3s /usr/local/bin/
$ chmod 755 /usr/local/bin/k3s
$ wget http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh

# 重启K3s服务
$ sudo systemctl restart k3s.service
```

### 1.5.2 启用自动升级功能[#](https://docs.rancher.cn/docs/k3s/installation/airgap/_index#启用自动升级功能)

除了可以通过脚本升级 K3s 以外，您还可以启用自动升级功能，以保持离线环境中的 K3s 版本与最新的 K3s 版本同步。

从 **v1.17.4+k3s1** 开始，K3s 支持[自动升级](https://docs.rancher.cn/docs/k3s/upgrades/automated/_index)。要在离线环境中启用此功能，您必须确保所需镜像在您的私有镜像仓库中可用。

1. 你将需要与你打算升级到的 K3s 版本相对应的 `rancher/k3s-upgrade` 版本。注意，镜像标签将 K3s 版本中的+替换为-，因为 Docker 镜像不支持+。
2. 你还需要在你要部署的`system-upgrad-controller manifest` YAML 中指定的 `system-upgrad-controller` 和`kubectl`的版本。在[这里](https://github.com/rancher/system-upgrade-controller/releases/latest)检查 `system-upgrad-controller` 的最新版本，并下载 `system-upgrad-controller.yaml` 来确定你需要推送到私有镜像仓库的版本。例如，在`system-upgrade-controller`的 v0.4.0 版本中，在 manifest YAML 中指定了这些镜像：

```shell
rancher/system-upgrade-controller:v0.4.0
rancher/kubectl:v0.17.0
```

1. 将必要的 `rancher/k3s-upgrade、rancher/system-upgrade-controller 和 rancher/kubectl` 镜像添加到您的私有镜像仓库中以后 ，就可以按照[K3s 自动升级指南](https://docs.rancher.cn/docs/k3s/upgrades/automated/_index)进行操作。