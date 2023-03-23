Video Reference：[03-安装-要求及选项](https://www.bilibili.com/video/BV1Gu411m7vx/?spm_id_from=333.788&vd_source=9560c118fae1db9638f05a6ba2527085)

GitHub README.md：[03-安装-要求及选项](https://github.com/kingsd041/k3s-tutorial/blob/main/03-安装-要求及选项/README.md)

## 1 安装要求

### 1.1 先决条件

- 两个节点不能有相同的主机名。

如果 k3s master 和 worker 节点主机名相同，注册 worker 节点时会报错：

```properties
# 造成这种错误的原因：
# 涉及到 K3s agent 的注册流程，首先如果想把K3s agent注册到K3s master中
# K3s agent首先会生成一串随机Password，K3s agent结合Password跟主机名Hostname传到K3s master
Apr 13 07:37:37 k3s2 k3s[1763]: time="2021-04-13T15:37:37.444123090+08:00" level=error msg="Failed to retrieve agent config: Node password rejected, duplicate hostname or contents of '/etc/rancher/node/password' may not match server node-passwd entry, try enabling a unique node name with the --with-node-id flag"
```

-  如果您的所有节点都有相同的主机名，请使用`--with-node-id`选项为每个节点添加一个随机后缀，或者为您添加到集群的每个节点设计一个独特的名称，用`--node-name`或`$K3S_NODE_NAME`传递。 

💡Tips

在没有明确设置 exec 命令的情况下设置`K3S_URL`，会将命令默认为 `"agent"`。

运行 agent 时还必须设置`K3S_TOKEN`。

- - `--with-node-id`：

```bash
# K3S_URL : 指定K3s URL
# K3S_TOKEN : 指定K3s Master Token[cat /var/lib/rancher/k3s/server/token]
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=https://10.0.0.30:6443 K3S_TOKEN=6f4f3e77082caf01f1e4874bbbab61a6 sh -s - --with-node-id
$ kubectl get nodes
NAME            STATUS   ROLES                  AGE   VERSION
k3s1            Ready    control-plane,master   67m   v1.25.6+k3s1
k3s2-3e0439f4   Ready    <none>                 15s   v1.25.6+k3s1
```

- - `指定主机名`：

```bash
# K3S_URL : 指定K3s Master URL
# K3S_TOKEN : 指定K3s Master Token[cat /var/lib/rancher/k3s/server/token]
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=https://10.0.0.30:6443 K3S_TOKEN=6f4f3e77082caf01f1e4874bbbab61a6 sh -s - --node-name k3s2
# K3S_URL : 指定K3s Master URL
# K3S_TOKEN : 指定K3s Master Token[cat /var/lib/rancher/k3s/server/token]
# 使用环境变量的方式指定主机名最后还是会以参数的形式进行修改
curl -sfL https://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | K3S_NODE_NAME="k3s2" INSTALL_K3S_MIRROR=cn K3S_URL=https://10.0.0.30:6443 K3S_TOKEN=6f4f3e77082caf01f1e4874bbbab61a6 sh -
$ kubectl get nodes
NAME   STATUS   ROLES                  AGE   VERSION
k3s1   Ready    control-plane,master   72m   v1.25.6+k3s1
k3s2   Ready    <none>                 28s   v1.25.6+k3s1
```

### 1.2 操作系统

K3s 有望在大多数现代 Linux 系统上运行:

- CentOS: 7.8, 7.9, 8.2, 8.3
- RHEL: 7.8, 7.9, 8.2, 8.3
- SLES: 15SP2
- Ubuntu: 16.04, 18.04, 20.04

### 1.3 硬件

硬件要求根据您部署的规模而变化。这里列出了最低建议:

- 内存： 最低 512MB（建议至少为 1GB）
- CPU： 最低 1

Reference：https://docs.rancher.cn/docs/k3s/installation/installation-requirements/resource-profiling/_index

| 组件                       | 处理器                                        | 最小 CPU      | Kine/SQLite 的最小内存 | 嵌入式 ETCD 的最小内存 |
| -------------------------- | --------------------------------------------- | ------------- | ---------------------- | ---------------------- |
| 具有工作负载的 K3s server  | Intel(R) Xeon(R) Platinum 8124M CPU, 3.00 GHz | 10% of a core | 768 M                  | 896 M                  |
| 具有单个 agent 的 K3s 集群 | Intel(R) Xeon(R) Platinum 8124M CPU, 3.00 GHz | 10% of a core | 512 M                  | 768 M                  |
| K3s agent                  | Intel(R) Xeon(R) Platinum 8124M CPU, 3.00 GHz | 5% of a core  | 256 M                  | 256 M                  |
| 具有工作负载的 K3s server  | Pi4B BCM2711, 1.50 GHz                        | 20% of a core | 768 M                  | 896 M                  |
| 具有单个 agent 的 K3s 集群 | Pi4B BCM2711, 1.50 GHz                        | 20% of a core | 512 M                  | 768 M                  |
| K3s agent                  | Pi4B BCM2711, 1.50 GHz                        | 10% of a core | 256 M                  | 256 M                  |

#### 1.3.1 磁盘

K3s 的性能取决于数据库的性能。为了确保最佳速度，我们建议尽可能使用 SSD。在使用 SD 卡或 eMMC 的 ARM 设备上，磁盘性能会有所不同。

### 1.4 网络

**K3s Server 节点的入站规则：**

| **协议** | **端口**  | **源**                   | **描述**                     |
| -------- | --------- | ------------------------ | ---------------------------- |
| TCP      | 6443      | K3s agent 节点           | Kubernetes API Server        |
| UDP      | 8472      | K3s server 和 agent 节点 | 仅对 Flannel VXLAN 需要      |
| TCP      | 10250     | K3s server 和 agent 节点 | Kubelet metrics              |
| TCP      | 2379-2380 | K3s server 节点          | 只有嵌入式 etcd 高可用才需要 |

**通常情况下，所有出站流量都是允许的。**

## 2 K3s 资源分析

### 2.1 影响资源利用率的因素

-  K3s server：K3s server 的利用率数据主要是由支持 Kubernetes 数据存储（kine 或 etcd）、API Server、Controller-Manager 和 Scheduler 控制。创建/修改/删除资源将导致暂时的利用率上升。大量使用 Kubernetes 数据存储的 operators 或应用程序也将增加 server 的资源需求。 
-  K3s agent：管理镜像、提供存储或创建/销毁容器的操作将导致利用率的暂时上升，拉取镜像通常会影响 CPU 和 IO，因为它们涉及将镜像内容解压到磁盘。如果可能的话，工作负载存储(pod 临时存储和卷)应该与 agent 组件(/var/lib/rancher/k3s/agent)隔离，以确保不会出现资源冲突。 

K3s server 的利用率数据主要是由支持 Kubernetes 数据存储（kine 或 etcd）、API Server、Controller-Manager 和 Scheduler 控制，以及实现系统状态变化所需的任何管理任务所驱动。给 Kubernetes 控制平面带来额外负载的操作，如创建/修改/删除资源，将导致暂时的利用率上升。使用大量使用 Kubernetes 数据存储的 operators 或应用程序（如 Rancher 或其他 operators 类型的应用程序）将增加 server 的资源需求。通过添加额外的节点或创建许多集群资源来扩展集群，将增加 server 的资源需求。

K3s agent 的利用率数据主要是由支持容器生命周期管理控制驱动的。涉及管理镜像、提供存储或创建/销毁容器的操作将导致利用率的暂时上升。拉取镜像通常会影响 CPU 和 IO，因为它们涉及将镜像内容解压到磁盘。如果可能的话，工作负载存储(pod 临时存储和卷)应该与 agent 组件(/var/lib/rancher/k3s/agent)隔离，以确保不会出现资源冲突。

### 2.2 防止 agent 和工作负载干扰集群数据存储

在 server 节点运行工作负载 pod 时，应确保 agent 和工作负载 IOPS 不干扰数据存储。

将 server 组件（/var/lib/rancher/k3s/server）与 agent 组件（/var/lib/rancher/k3s/agent）放在不同的存储介质上，后者包括 containerd 镜像存储。

工作负载存储（pod 临时存储和卷）也应该与数据存储隔离。

## 3 安装选项

### 3.1 使用脚本安装的选项

| **Environment Variable**        | **Description**                                              |
| ------------------------------- | ------------------------------------------------------------ |
| `INSTALL_K3S_SKIP_DOWNLOAD`     | 如果设置为 "true "将不会下载 K3s 的哈希值或二进制。          |
| `INSTALL_K3S_SYMLINK`           | 默认情况下，如果路径中不存在命令，将为 kubectl、crictl 和 ctr 二进制文件创建符号链接。如果设置为'skip'将不会创建符号链接，而'force'将覆盖。 |
| `INSTALL_K3S_SKIP_ENABLE`       | 如果设置为 "true"，将不启用或启动 K3s 服务。                 |
| `INSTALL_K3S_SKIP_START`        | 如果设置为 "true "将不会启动 K3s 服务。                      |
| `INSTALL_K3S_VERSION`           | 从 Github 下载 K3s 的版本。如果没有指定，将尝试从"stable"频道下载。 |
| `INSTALL_K3S_BIN_DIR`           | 安装 K3s 二进制文件、链接和卸载脚本的目录，或者使用/usr/local/bin作为默认目录。 |
| `INSTALL_K3S_BIN_DIR_READ_ONLY` | 如果设置为 true 将不会把文件写入INSTALL_K3S_BIN_DIR，强制设置INSTALL_K3S_SKIP_DOWNLOAD=true。 |
| `INSTALL_K3S_SYSTEMD_DIR`       | 安装 systemd 服务和环境文件的目录，或者使用/etc/systemd/system作为默认目录。 |
| `INSTALL_K3S_EXEC`              | 带有标志的命令，用于在服务中启动 K3s。如果未指定命令，并且设置了K3S_URL，它将默认为“agent”。如果未设置K3S_URL，它将默认为“server”。要获得帮助，请参考[此示例。](https://docs.rancher.cn/docs/k3s/installation/install-options/how-to-flags/_index#示例-b-install_k3s_exec) |
| `INSTALL_K3S_NAME`              | 要创建的 systemd 服务名称，如果以服务器方式运行 k3s，则默认为'k3s'；如果以 agent 方式运行 k3s，则默认为'k3s-agent'。如果指定了服务名，则服务名将以'k3s-'为前缀。 |
| `INSTALL_K3S_TYPE`              | 要创建的 systemd 服务类型，如果没有指定，将默认使用 K3s exec 命令。 |
| `INSTALL_K3S_SELINUX_WARN`      | 如果设置为 true，则在没有找到 k3s-selinux 策略的情况下将继续。 |
| `INSTALL_K3S_SKIP_SELINUX_RPM`  | 如果设置为 "true "将跳过 k3s RPM 的自动安装。                |
| `INSTALL_K3S_CHANNEL_URL`       | 用于获取 K3s 下载网址的频道 URL。默认为 https://update.k3s.io/v1-release/channels。 |
| `INSTALL_K3S_CHANNEL`           | 用于获取 K3s 下载 URL 的通道。默认值为 "stable"。选项包括：stable, latest, testing。 |
| `K3S_CONFIG_FILE`               | 指定配置文件的位置。默认目录为/etc/rancher/k3s/config.yaml。 |
| `K3S_TOKEN`                     | 用于将 server 或 agent 加入集群的共享 secret。               |
| `K3S_TOKEN_FILE`                | 指定 cluster-secret,token 的文件目录。                       |

- 默认的启动过程

```shell
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -
# 默认使用stable的kubernetes版本
[INFO]  Finding release for channel stable
# 找到v1.25.6+k3s1的K3s版本
[INFO]  Using v1.25.6+k3s1 as release
# 使用国内的链接下载Hash文件
[INFO]  Downloading hash rancher-mirror.rancher.cn/k3s/v1.25.6-k3s1/sha256sum-amd64.txt
# 以及从国内下载k3s二进制文件
[INFO]  Downloading binary rancher-mirror.rancher.cn/k3s/v1.25.6-k3s1/k3s
[INFO]  Verifying binary download
# 创建三个符号链接
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
# 创建两个脚本文件
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh # 目的为了杀掉K3s进程
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh # 目的为了卸载K3s集群
# 创建K3s的service文件
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
# 启动K3s集群
[INFO]  systemd: Enabling k3s unit
Created symlink from /etc/systemd/system/multi-user.target.wants/k3s.service to /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s
```

-  `INSTALL_K3S_SKIP_DOWNLOAD` -- (用于离线安装) 如果设置为 "true "将不会下载 K3s 的哈希值或二进制。 

```bash
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_SKIP_DOWNLOAD=true \
  sh -
[INFO]  Skipping k3s download and verify
[ERROR]  Executable k3s binary not found at /usr/local/bin/k3s

# 下载k3s二进制文件
$ wget https://rancher-mirror.rancher.cn/k3s/v1.25.6-k3s1/k3s
--2021-04-14 15:57:27--  http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
Resolving rancher-mirror.cnrancher.com (rancher-mirror.cnrancher.com)... 59.110.191.1
Connecting to rancher-mirror.cnrancher.com (rancher-mirror.cnrancher.com)|59.110.191.1|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 45387776 (43M) [application/octet-stream]
Saving to: ‘k3s’

k3s                                             100%[======================================================================================================>]  43.29M  4.61MB/s    in 11s

2021-04-14 15:57:38 (4.09 MB/s) - ‘k3s’ saved [45387776/45387776]

$ chmod a+x k3s && mv k3s /usr/local/bin/
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_SKIP_DOWNLOAD=true \
  sh -
[INFO]  Skipping k3s download and verify
[INFO]  Skipping installation of SELinux RPM
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink from /etc/systemd/system/multi-user.target.wants/k3s.service to /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s

$ ls -l /usr/local/bin/
total 65188
lrwxrwxrwx 1 root root        3 Mar  4 13:25 crictl -> k3s
-rwxr-xr-x 1 root root 66744320 Jan 26 12:28 k3s
-rwxr-xr-x 1 root root     2026 Mar  4 13:25 k3s-killall.sh
-rwxr-xr-x 1 root root     1397 Mar  4 13:25 k3s-uninstall.sh
lrwxrwxrwx 1 root root        3 Mar  4 13:25 kubectl -> k3s
```

-  `INSTALL_K3S_SYMLINK` -- 默认情况下，如果路径中不存在命令，将为 kubectl、crictl 和 ctr 二进制文件创建符号链接。如果设置为'`skip`'将不会创建符号链接，而'`force`'将覆盖。 

```bash
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_SYMLINK=skip \
  sh -
[INFO]  Finding release for channel stable
[INFO]  Using v1.25.6+k3s1 as release
[INFO]  Downloading hash rancher-mirror.rancher.cn/k3s/v1.25.6-k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary rancher-mirror.rancher.cn/k3s/v1.25.6-k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink from /etc/systemd/system/multi-user.target.wants/k3s.service to /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s

# 没有创建出符号链接
$ ls -l /usr/local/bin/
total 65188
-rwxr-xr-x 1 root root 66744320 Mar  4 13:30 k3s
-rwxr-xr-x 1 root root     2026 Mar  4 13:30 k3s-killall.sh
-rwxr-xr-x 1 root root     1397 Mar  4 13:30 k3s-uninstall.sh
```

-  `INSTALL_K3S_SKIP_ENABLE` -- 如果设置为 "`true`"，将不启用或启动 K3s 服务。 该环境变量使用较少。

```bash
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_SKIP_ENABLE=true \
  sh -
[INFO]  Finding release for channel stable
[INFO]  Using v1.25.6+k3s1 as release
[INFO]  Downloading hash rancher-mirror.rancher.cn/k3s/v1.25.6-k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary rancher-mirror.rancher.cn/k3s/v1.25.6-k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink from /etc/systemd/system/multi-user.target.wants/k3s.service to /etc/systemd/system/k3s.service.

$ systemctl enable /etc/systemd/system/k3s.service
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
```

-  `INSTALL_K3S_SKIP_START` -- 如果设置为 "`true` "将不会启动 K3s 服务。  该环境变量使用较少。

```bash
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_SKIP_START=true \
  sh -
[INFO]  Finding release for channel stable
[INFO]  Using v1.20.5+k3s1 as release
[INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit

$ systemctl start k3s.service
```

-  `INSTALL_K3S_VERSION` -- 从 Github 下载 K3s 的版本。如果没有指定，将尝试从"`stable`"频道下载。 

```bash
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_VERSION="v1.26.1+k3s1" \
  sh -
[INFO]  Using v1.26.1+k3s1 as release
[INFO]  Downloading hash rancher-mirror.rancher.cn/k3s/v1.26.1-k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary rancher-mirror.rancher.cn/k3s/v1.26.1-k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink from /etc/systemd/system/multi-user.target.wants/k3s.service to /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s

$ kubectl get nodes
NAME   STATUS   ROLES                  AGE   VERSION
k3s1   Ready    control-plane,master   40s   v1.26.1+k3s1
```

需要在GitHub中查看相应的版本号支持

Reference：https://github.com/k3s-io/k3s

-  `INSTALL_K3S_BIN_DIR` -- 安装 K3s 二进制文件、链接和卸载脚本的目录，或者使用`/usr/local/bin`作为默认目录。该环境变量使用较少。

```bash
$ mkdir -pv /opt/bin/
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_BIN_DIR=/opt/bin \
  sh -
[INFO]  Finding release for channel stable
[INFO]  Using v1.25.6+k3s1 as release
[INFO]  Downloading hash rancher-mirror.rancher.cn/k3s/v1.25.6-k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary rancher-mirror.rancher.cn/k3s/v1.25.6-k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /opt/bin/k3s
[INFO]  Creating /opt/bin/kubectl symlink to k3s
[INFO]  Creating /opt/bin/crictl symlink to k3s
[INFO]  Skipping /opt/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
[INFO]  Creating killall script /opt/bin/k3s-killall.sh
[INFO]  Creating uninstall script /opt/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink from /etc/systemd/system/multi-user.target.wants/k3s.service to /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s

$ ls -l /opt/bin/
total 65188
lrwxrwxrwx 1 root root        3 Mar  4 13:42 crictl -> k3s
-rwxr-xr-x 1 root root 66744320 Mar  4 13:41 k3s
-rwxr-xr-x 1 root root     2026 Mar  4 13:42 k3s-killall.sh
-rwxr-xr-x 1 root root     1361 Mar  4 13:42 k3s-uninstall.sh
lrwxrwxrwx 1 root root        3 Mar  4 13:42 kubectl -> k3s
```

-  `INSTALL_K3S_BIN_DIR_READ_ONLY` -- 如果设置为 true 将不会把文件写入`INSTALL_K3S_BIN_DIR`，强制设置`INSTALL_K3S_SKIP_DOWNLOAD=true`。`INSTALL_K3S_SKIP_DOWNLOAD` 会创建`kubectl/crictl/ctr`等，而 `INSTALL_K3S_BIN_DIR_READ_ONLY` 不创建。该环境变量使用较少。

```bash
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_BIN_DIR_READ_ONLY=true \
  sh -
[INFO]  Skipping k3s download and verify
[ERROR]  Executable k3s binary not found at /usr/local/bin/k3s

$ cp k3s /usr/local/bin/
$ curl -sfL hhttps://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_BIN_DIR_READ_ONLY=true \
  sh -
[INFO]  Skipping k3s download and verify
[INFO]  Skipping installation of SELinux RPM
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
[INFO]  systemd: Starting k3s

$ ls /usr/local/bin/
k3s
```

-  `INSTALL_K3S_SYSTEMD_DIR` -- 安装 systemd 服务和环境文件的目录，或者使用`/etc/systemd/system` 作为默认目录。该环境变量使用较少。

```bash
$ mkdir -pv /opt/systemd
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_SYSTEMD_DIR=/opt/systemd \
  sh -
[INFO]  Finding release for channel stable
[INFO]  Using v1.20.5+k3s1 as release
[INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /opt/systemd/k3s.service.env
[INFO]  systemd: Creating service file /opt/systemd/k3s.service
[INFO]  systemd: Enabling k3s unit
[INFO]  systemd: Starting k3s
```

-  `INSTALL_K3S_EXEC` -- 带有标志的命令，用于在服务中启动 K3s。如果未指定命令，并且设置了 K3S_URL，它将默认为“`agent`”。如果未设置 `K3S_URL`，它将默认为“`server`”。 

```bash
# 使用 Docker 作为容器运行时环境
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC="--docker" \
  sh -
[INFO]  Finding release for channel stable
[INFO]  Using v1.25.6+k3s1 as release
[INFO]  Downloading hash rancher-mirror.rancher.cn/k3s/v1.25.6-k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary rancher-mirror.rancher.cn/k3s/v1.25.6-k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink from /etc/systemd/system/multi-user.target.wants/k3s.service to /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s

# 查看Docker启动的容器
$ docker ps -a
```

-  `INSTALL_K3S_NAME` -- 要创建的 systemd 服务名称，如果以服务器方式运行 k3s，则默认为'k3s'；如果以 agent 方式运行 k3s，则默认为'k3s-agent'。如果指定了服务名，则服务名将以'k3s-'为前缀。 

```bash
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_NAME="demo" \
  sh -
[INFO]  Finding release for channel stable
[INFO]  Using v1.20.5+k3s1 as release
[INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-demo-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s-demo.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s-demo.service
[INFO]  systemd: Enabling k3s-demo unit
[INFO]  systemd: Starting k3s-demo

$ systemctl status k3s-demo
```

-  `INSTALL_K3S_TYPE` -- 要创建的 systemd 服务类型，默认为 notify。该环境变量使用较少。

```bash
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_TYPE="exec" \
  sh -
[INFO]  Finding release for channel stable
[INFO]  Using v1.20.5+k3s1 as release
[INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
[INFO]  systemd: Starting k3s

$ cat /etc/systemd/system/k3s.service
[Unit]
Description=Lightweight Kubernetes
Documentation=https://k3s.io
Wants=network-online.target
After=network-online.target
[Install]
WantedBy=multi-user.target
[Service]
Type=exec
EnvironmentFile=/etc/systemd/system/k3s.service.env
KillMode=process
Delegate=yes
```

-  `INSTALL_K3S_SKIP_SELINUX_RPM` -- 如果设置为 "`true` "将跳过 k3s RPM 的自动安装。 

```bash
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_SKIP_SELINUX_RPM=true \
  sh -
[INFO]  Finding release for channel stable
[INFO]  Using v1.20.5+k3s1 as release
[INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Skipping installation of SELinux RPM
[ERROR]  Failed to find the k3s-selinux policy, please install:
    yum install -y container-selinux selinux-policy-base
    yum install -y https://rpm.rancher.io/k3s/stable/common/centos/7/noarch/k3s-selinux-0.2-1.el7_8.noarch.rpm
```

-  `INSTALL_K3S_CHANNEL_URL` -- 用于获取 K3s 下载网址的频道 URL。默认为 https://update.k3s.io/v1-release/channels 。 
-  `INSTALL_K3S_CHANNEL` -- 用于获取 K3s 下载 URL 的通道。默认值为 "`stable`"。选项包括：`stable`, `latest`, `testing`。 

```bash
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_CHANNEL="latest" \
  sh -
[INFO]  Finding release for channel latest
[INFO]  Using v1.20.5+k3s1 as release
[INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
[INFO]  systemd: Starting k3s
```

-  `K3S_CONFIG_FILE` -- 指定配置文件的位置。默认目录为`/etc/rancher/k3s/config.yaml`。 该环境变量使用较少。

```bash
$ cat <<-'EOF' > /opt/config.yaml
node-label:
- "foo=bar"
- "something=amazing"
EOF

$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  K3S_CONFIG_FILE=/opt/config.yaml \
  sh -
[INFO]  Finding release for channel stable
[INFO]  Using v1.25.6+k3s1 as release
[INFO]  Downloading hash rancher-mirror.rancher.cn/k3s/v1.25.6-k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary rancher-mirror.rancher.cn/k3s/v1.25.6-k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink from /etc/systemd/system/multi-user.target.wants/k3s.service to /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s

$ kubectl get nodes --show-labels
NAME   STATUS   ROLES                  AGE   VERSION        LABELS
k3s1   Ready    control-plane,master   81s   v1.25.6+k3s1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=k3s,beta.kubernetes.io/os=linux,egress.k3s.io/cluster=true,foo=bar,kubernetes.io/arch=amd64,kubernetes.io/hostname=k3s2,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=true,node-role.kubernetes.io/master=true,node.kubernetes.io/instance-type=k3s,something=amazing
```

-  `K3S_TOKEN` -- 用于将 `server` 或 `agent` 加入集群的共享 `secret`。 在自动化环境中使用较多。

```bash
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  K3S_TOKEN=rancher-k3s \
  sh -
[INFO]  Finding release for channel stable
[INFO]  Using v1.20.5+k3s1 as release
[INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
[INFO]  systemd: Starting k3s

$ cat /var/lib/rancher/k3s/server/token
K1042465c14be8de6a57c482b4162f673addcb652acb13c8119a9900b5d27c234f7::server:rancher-k3s
```

-  `K3S_TOKEN_FILE` -- 指定 `cluster-secret`,`token` 的文件目录。 和`K3S_TOKEN`大同小异。

```bash
$ cat /opt/token.txt
rancher-k3s

$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  K3S_TOKEN_FILE=/opt/token.txt \
  sh -
[INFO]  Finding release for channel stable
[INFO]  Using v1.20.5+k3s1 as release
[INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
[INFO]  systemd: Starting k3s

$ cat /var/lib/rancher/k3s/server/token
K10d7a2e502103611961b4c6ab18baac06beb34f606886698c905f346671d6c432f::server:rancher-k3s
```

#### 3.1.1 其他说明

1. 以 "`K3S_`"开头的环境变量将被保留，供 `systemd` 和 `openrc` 服务使用。
2. 在没有明确设置 `exec` 命令的情况下设置 `K3S_URL`，会将命令默认为 "`agent`"。
3. 运行 `agent` 时还必须设置 `K3S_TOKEN`。

### 3.2 从二进制中安装 K3s

安装脚本主要是配置 K3s 作为服务运行。如果你选择不使用脚本，你可以通过从我们的[发布页面](https://github.com/rancher/k3s/releases/latest)下载二进制文件，将其放在你的路径上，然后执行它来运行 K3s。K3s 二进制支持以下命令：

| **命令**      | **描述**                                                     |
| ------------- | ------------------------------------------------------------ |
| `k3s server`  | 运行 K3s server，它还将启动 Kubernetes control-plane 组件，如 API server, controller-manager, 和 scheduler。 |
| `k3s agent`   | 运行 K3s agent 节点。这将使 K3s 作为工作节点运行，启动 Kubernetes 节点服务kubelet和kube-proxy。 |
| `k3s kubectl` | 运行嵌入式[kubectl](https://kubernetes.io/docs/reference/kubectl/overview/) CLI。如果没有设置KUBECONFIG环境变量，当启动 K3s 服务器节点时，将自动尝试使用在/etc/rancher/k3s/k3s.yaml创建的配置文件。 |
| `k3s crictl`  | 运行一个嵌入式[crictl](https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md)。这是一个用于与 Kubernetes 的容器运行时接口（CRI）交互的 CLI。对调试很有用。 |
| `k3s ctr`     | 运行一个嵌入式的[ctr](https://github.com/projectatomic/containerd/blob/master/docs/cli.md)。这是为 containerd（K3s 使用的容器守护进程）提供的 CLI。对调试很有用。 |
| `k3s help`    | 显示一个命令的命令列表或帮助。                               |

k3s server 和 k3s agent 命令有额外的配置选项，可以通过 k3s server --help 或 k3s agent --help 查看.

```shell
$ wget https://github.com/k3s-io/k3s/releases/download/v1.26.1%2Bk3s1/k3s -P /usr/loca/bin/
```

-  `k3s server` -- 运行 K3s server，它还将启动 Kubernetes control-plane 组件，如 API server, controller-manager, 和 scheduler。 

```bash
$ chmod +x /usr/local/bin/k3s
$ k3s server
INFO[0000] Acquiring lock file /var/lib/rancher/k3s/data/.lock
INFO[0000] Preparing data dir /var/lib/rancher/k3s/data/640b9915ffbb627ea4a2213ce03a1861e755a23967f45ab7241c094e10a6749f
INFO[0000] Starting k3s v1.26.1+k3s1 (f10af367)
[......]

$ kubectl
bash: kubectl: command not found...
$ k3s kubectl get nodes
NAME   STATUS   ROLES                  AGE   VERSION
k3s2   Ready    control-plane,master   72s   v1.26.1+k3s1
```

-  `k3s agent` -- 运行 K3s agent 节点。这将使 K3s 作为工作节点运行，启动 Kubernetes 节点服务 kubelet 和 kube-proxy。 

```bash
$ k3s agent --server https://192.168.64.3:6443 --token 3c9c7aebc5cd83fb1ad92c5a7686eb95
INFO[0000] Acquiring lock file /var/lib/rancher/k3s/data/.lock
INFO[0000] Preparing data dir /var/lib/rancher/k3s/data/926943070e893920b703e893777d0cdc577dea7609f819abb14a97ea8ea1d30b
INFO[2021-04-15T10:09:57.962476104+08:00] Starting k3s agent v1.20.5+k3s1 (355fff30)
INFO[2021-04-15T10:09:58.351011343+08:00] Running load balancer 127.0.0.1:35105 -[192.168.64.3:6443]
[......]
```

-  `k3s kubectl` -- 运行嵌入式 kubectl CLI。如果没有设置 KUBECONFIG 环境变量，当启动 K3s 服务器节点时，将自动尝试使用在/etc/rancher/k3s/k3s.yaml 创建的配置文件。 

```bash
$ k3s kubectl get nodes
NAME   STATUS   ROLES                  AGE     VERSION
k3s1   Ready    control-plane,master   8m14s   v1.20.5+k3s1
k3s2   Ready    <none                11s     v1.20.5+k3s1
```

-  `k3s crictl` -- 运行一个嵌入式 crictl。这是一个用于与 Kubernetes 的容器运行时接口（CRI）交互的 CLI。对调试很有用。 

```bash
$ k3s crictl ps
CONTAINER           IMAGE               CREATED             STATE               NAME                     ATTEMPT             POD ID
9ceb610df16c7       aa764f7db3051       8 minutes ago       Running             traefik                  0                   373c79416fa65
cadceb62ae08d       897ce3c5fc8ff       8 minutes ago       Running             lb-port-443              0                   f8a0ecfe56562
a26a49be485ac       897ce3c5fc8ff       8 minutes ago       Running             lb-port-80               0                   f8a0ecfe56562
01894072f2298       148c192562719       8 minutes ago       Running             local-path-provisioner   1                   b9d55e63f632f
5ccd6ed05120f       296a6d5035e2d       9 minutes ago       Running             coredns                  0                   2bae007d8e486
be0765e77a703       9dd718864ce61       9 minutes ago       Running             metrics-server           0                   53ab949c026ce
```

-  `k3s ctr` -- 运行一个嵌入式的 ctr。这是为 containerd（K3s 使用的容器守护进程）提供的 CLI。对调试很有用。 

```bash
$ k3s ctr container ls
CONTAINER                                                           IMAGE                                                                                                               RUNTIME
01894072f2298208f3c109f9fb1d5e12e677d11cd5d0b0a3a66f550ae38644e4    docker.io/rancher/local-path-provisioner:v0.0.19                                                                    io.containerd.runc.v2
2bae007d8e486afffbbf1ffb88e97b92d367aff4b06842217de4fb5d22ecf1b9    docker.io/rancher/pause:3.1
```

`k3s server` 和 `k3s agent` 命令有额外的配置选项，可以通过 `k3s server --help` 或 `k3s agent --help` 查看.

### 3.3 通过配置文件启动 K3s

除了使用环境变量和 CLI 参数来配置 K3s，K3s 还可以使用配置文件。默认目录位于`/etc/rancher/k3s/config.yaml`

```bash
$ mkdir -pv /etc/rancher/k3s
$ cat /etc/rancher/k3s/config.yaml
node-label:
- "k3s-role=master"

$ k3s server
INFO[2021-04-15T10:19:45.651637303+08:00] Starting k3s v1.20.5+k3s1 (355fff30)
INFO[2021-04-15T10:19:45.652149355+08:00] Cluster bootstrap already complete
INFO[2021-04-15T10:19:45.666066590+08:00] Configuring sqlite3 database connection pooling: maxIdleConns=2, maxOpenConns=0, connMaxLifetime=0s
INFO[2021-04-15T10:19:45.666263783+08:00] Configuring database table schema and indexes, this may take a moment...

$ k3s kubectl get nodes --show-labels
NAME   STATUS   ROLES                  AGE   VERSION        LABELS
k3s1   Ready    control-plane,master   88s   v1.20.5+k3s1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=k3s,beta.kubernetes.io/os=linux,k3s-role=master,k3s-rule=master,k3s.io/hostname=k3s1,k3s.io/internal-ip=192.168.64.3,kubernetes.io/arch=amd64,kubernetes.io/hostname=k3s1,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=true,node-role.kubernetes.io/master=true,node.kubernetes.io/instance-type=k3s
```

如果同时使用配置文件和 CLI 参数。 在这种情况下，值将从两个来源加载，但 CLI 参数优先级更高。 对于可重复的参数，如`--node-label`，CLI 参数将覆盖列表中的所有值。

## 3.4 Reference

```shell
#!/bin/sh
set -e
set -o noglob

# Usage:
#   curl ... | ENV_VAR=... sh -
#       or
#   ENV_VAR=... ./install.sh
#
# Example:
#   Installing a server without traefik:
#     curl ... | INSTALL_K3S_EXEC="--disable=traefik" sh -
#   Installing an agent to point at a server:
#     curl ... | K3S_TOKEN=xxx K3S_URL=https://server-url:6443 sh -
#
# Environment variables:
#   - K3S_*
#     Environment variables which begin with K3S_ will be preserved for the
#     systemd service to use. Setting K3S_URL without explicitly setting
#     a systemd exec command will default the command to "agent", and we
#     enforce that K3S_TOKEN is also set.
#
#   - INSTALL_K3S_SKIP_DOWNLOAD
#     If set to true will not download k3s hash or binary.
#
#   - INSTALL_K3S_FORCE_RESTART
#     If set to true will always restart the K3s service
#
#   - INSTALL_K3S_SYMLINK
#     If set to 'skip' will not create symlinks, 'force' will overwrite,
#     default will symlink if command does not exist in path.
#
#   - INSTALL_K3S_SKIP_ENABLE
#     If set to true will not enable or start k3s service.
#
#   - INSTALL_K3S_SKIP_START
#     If set to true will not start k3s service.
#
#   - INSTALL_K3S_VERSION
#     Version of k3s to download from github. Will attempt to download from the
#     stable channel if not specified.
#
#   - INSTALL_K3S_COMMIT
#     Commit of k3s to download from temporary cloud storage.
#     * (for developer & QA use)
#
#   - INSTALL_K3S_BIN_DIR
#     Directory to install k3s binary, links, and uninstall script to, or use
#     /usr/local/bin as the default
#
#   - INSTALL_K3S_BIN_DIR_READ_ONLY
#     If set to true will not write files to INSTALL_K3S_BIN_DIR, forces
#     setting INSTALL_K3S_SKIP_DOWNLOAD=true
#
#   - INSTALL_K3S_SYSTEMD_DIR
#     Directory to install systemd service and environment files to, or use
#     /etc/systemd/system as the default
#
#   - INSTALL_K3S_EXEC or script arguments
#     Command with flags to use for launching k3s in the systemd service, if
#     the command is not specified will default to "agent" if K3S_URL is set
#     or "server" if not. The final systemd command resolves to a combination
#     of EXEC and script args ($@).
#
#     The following commands result in the same behavior:
#       curl ... | INSTALL_K3S_EXEC="--disable=traefik" sh -s -
#       curl ... | INSTALL_K3S_EXEC="server --disable=traefik" sh -s -
#       curl ... | INSTALL_K3S_EXEC="server" sh -s - --disable=traefik
#       curl ... | sh -s - server --disable=traefik
#       curl ... | sh -s - --disable=traefik
#
#   - INSTALL_K3S_NAME
#     Name of systemd service to create, will default from the k3s exec command
#     if not specified. If specified the name will be prefixed with 'k3s-'.
#
#   - INSTALL_K3S_TYPE
#     Type of systemd service to create, will default from the k3s exec command
#     if not specified.
#
#   - INSTALL_K3S_MIRROR
#     For Chinese users, set INSTALL_K3S_MIRROR=cn to use the mirror address to accelerate
#     k3s binary file download, and the default mirror address is rancher-mirror.rancher.cn
#
#   - INSTALL_K3S_SELINUX_WARN
#     If set to true will continue if k3s-selinux policy is not found.
#
#   - INSTALL_K3S_SKIP_SELINUX_RPM
#     If set to true will skip automatic installation of the k3s RPM.
#
#   - INSTALL_K3S_CHANNEL_URL
#     Channel URL for fetching k3s download URL.
#     Defaults to 'https://update.k3s.io/v1-release/channels'.
#
#   - INSTALL_K3S_CHANNEL
#     Channel to use for fetching k3s download URL.
#     Defaults to 'stable'.
#
#   - INSTALL_K3S_REGISTRIES
#     Setup a custom Registry or Mirror
#     Defaults to null.

GITHUB_URL=https://github.com/k3s-io/k3s/releases
STORAGE_URL=https://k3s-ci-builds.s3.amazonaws.com
DOWNLOADER=
INSTALL_K3S_MIRROR_URL=${INSTALL_K3S_MIRROR_URL:-'rancher-mirror.rancher.cn'}

# --- helper functions for logs ---
info()
{
    echo '[INFO] ' "$@"
}
warn()
{
    echo '[WARN] ' "$@" >&2
}
fatal()
{
    echo '[ERROR] ' "$@" >&2
    exit 1
}

# --- fatal if no systemd or openrc ---
verify_system() {
    if [ -x /sbin/openrc-run ]; then
        HAS_OPENRC=true
        return
    fi
    if [ -x /bin/systemctl ] || type systemctl > /dev/null 2>&1; then
        HAS_SYSTEMD=true
        return
    fi
    fatal 'Can not find systemd or openrc to use as a process supervisor for k3s'
}

# --- add quotes to command arguments ---
quote() {
    for arg in "$@"; do
        printf '%s\n' "$arg" | sed "s/'/'\\\\''/g;1s/^/'/;\$s/\$/'/"
    done
}

# --- add indentation and trailing slash to quoted args ---
quote_indent() {
    printf ' \\\n'
    for arg in "$@"; do
        printf '\t%s \\\n' "$(quote "$arg")"
    done
}

# --- escape most punctuation characters, except quotes, forward slash, and space ---
escape() {
    printf '%s' "$@" | sed -e 's/\([][!#$%&()*;<=>?\_`{|}]\)/\\\1/g;'
}

# --- escape double quotes ---
escape_dq() {
    printf '%s' "$@" | sed -e 's/"/\\"/g'
}

# --- ensures $K3S_URL is empty or begins with https://, exiting fatally otherwise ---
verify_k3s_url() {
    case "${K3S_URL}" in
        "")
            ;;
        https://*)
            ;;
        *)
            fatal "Only https:// URLs are supported for K3S_URL (have ${K3S_URL})"
            ;;
    esac
}

# --- Setup a custom Registry or Mirror
setup_registry() {
    REGISTRIES_FILE="/etc/rancher/k3s/registries.yaml"
    if [ "${INSTALL_K3S_REGISTRIES}" -a ! -f "$REGISTRIES_FILE" ]; then
        INSTALL_K3S_REGISTRIES=`echo ${INSTALL_K3S_REGISTRIES} | awk '{gsub(/,/," "); print $0}'`
        $SUDO mkdir -p `dirname $REGISTRIES_FILE`
        $SUDO cat >> $REGISTRIES_FILE <<EOF
mirrors:
  "docker.io":
    endpoint:
EOF
        for registry in ${INSTALL_K3S_REGISTRIES}; do
            echo "      - $registry" >> "$REGISTRIES_FILE"
        done
    fi
}

# --- define needed environment variables ---
setup_env() {
    # --- use command args if passed or create default ---
    case "$1" in
        # --- if we only have flags discover if command should be server or agent ---
        (-*|"")
            if [ -z "${K3S_URL}" ]; then
                CMD_K3S=server
            else
                if [ -z "${K3S_TOKEN}" ] && [ -z "${K3S_TOKEN_FILE}" ]; then
                    fatal "Defaulted k3s exec command to 'agent' because K3S_URL is defined, but K3S_TOKEN or K3S_TOKEN_FILE is not defined."
                fi
                CMD_K3S=agent
            fi
        ;;
        # --- command is provided ---
        (*)
            CMD_K3S=$1
            shift
        ;;
    esac

    verify_k3s_url

    CMD_K3S_EXEC="${CMD_K3S}$(quote_indent "$@")"

    # --- use systemd name if defined or create default ---
    if [ -n "${INSTALL_K3S_NAME}" ]; then
        SYSTEM_NAME=k3s-${INSTALL_K3S_NAME}
    else
        if [ "${CMD_K3S}" = server ]; then
            SYSTEM_NAME=k3s
        else
            SYSTEM_NAME=k3s-${CMD_K3S}
        fi
    fi

    # --- check for invalid characters in system name ---
    valid_chars=$(printf '%s' "${SYSTEM_NAME}" | sed -e 's/[][!#$%&()*;<=>?\_`{|}/[:space:]]/^/g;' )
    if [ "${SYSTEM_NAME}" != "${valid_chars}"  ]; then
        invalid_chars=$(printf '%s' "${valid_chars}" | sed -e 's/[^^]/ /g')
        fatal "Invalid characters for system name:
            ${SYSTEM_NAME}
            ${invalid_chars}"
    fi

    # --- use sudo if we are not already root ---
    SUDO=sudo
    if [ $(id -u) -eq 0 ]; then
        SUDO=
    fi

    # --- use systemd type if defined or create default ---
    if [ -n "${INSTALL_K3S_TYPE}" ]; then
        SYSTEMD_TYPE=${INSTALL_K3S_TYPE}
    else
        SYSTEMD_TYPE=notify
    fi

    # --- use binary install directory if defined or create default ---
    if [ -n "${INSTALL_K3S_BIN_DIR}" ]; then
        BIN_DIR=${INSTALL_K3S_BIN_DIR}
    else
        # --- use /usr/local/bin if root can write to it, otherwise use /opt/bin if it exists
        BIN_DIR=/usr/local/bin
        if ! $SUDO sh -c "touch ${BIN_DIR}/k3s-ro-test && rm -rf ${BIN_DIR}/k3s-ro-test"; then
            if [ -d /opt/bin ]; then
                BIN_DIR=/opt/bin
            fi
        fi
    fi

    # --- use systemd directory if defined or create default ---
    if [ -n "${INSTALL_K3S_SYSTEMD_DIR}" ]; then
        SYSTEMD_DIR="${INSTALL_K3S_SYSTEMD_DIR}"
    else
        SYSTEMD_DIR=/etc/systemd/system
    fi

    # --- set related files from system name ---
    SERVICE_K3S=${SYSTEM_NAME}.service
    UNINSTALL_K3S_SH=${UNINSTALL_K3S_SH:-${BIN_DIR}/${SYSTEM_NAME}-uninstall.sh}
    KILLALL_K3S_SH=${KILLALL_K3S_SH:-${BIN_DIR}/k3s-killall.sh}

    # --- use service or environment location depending on systemd/openrc ---
    if [ "${HAS_SYSTEMD}" = true ]; then
        FILE_K3S_SERVICE=${SYSTEMD_DIR}/${SERVICE_K3S}
        FILE_K3S_ENV=${SYSTEMD_DIR}/${SERVICE_K3S}.env
    elif [ "${HAS_OPENRC}" = true ]; then
        $SUDO mkdir -p /etc/rancher/k3s
        FILE_K3S_SERVICE=/etc/init.d/${SYSTEM_NAME}
        FILE_K3S_ENV=/etc/rancher/k3s/${SYSTEM_NAME}.env
    fi

    # --- get hash of config & exec for currently installed k3s ---
    PRE_INSTALL_HASHES=$(get_installed_hashes)

    # --- if bin directory is read only skip download ---
    if [ "${INSTALL_K3S_BIN_DIR_READ_ONLY}" = true ]; then
        INSTALL_K3S_SKIP_DOWNLOAD=true
    fi

    # --- setup channel values
    if [ "${INSTALL_K3S_MIRROR}" = cn ]; then
        INSTALL_K3S_CHANNEL_URL="${INSTALL_K3S_MIRROR_URL}/k3s/channels"
    else
        INSTALL_K3S_CHANNEL_URL=${INSTALL_K3S_CHANNEL_URL:-'https://update.k3s.io/v1-release/channels'}
    fi
    INSTALL_K3S_CHANNEL=${INSTALL_K3S_CHANNEL:-'stable'}
}

# --- check if skip download environment variable set ---
can_skip_download_binary() {
    if [ "${INSTALL_K3S_SKIP_DOWNLOAD}" != true ] && [ "${INSTALL_K3S_SKIP_DOWNLOAD}" != binary ]; then
        return 1
    fi
}

can_skip_download_selinux() {                                                        
    if [ "${INSTALL_K3S_SKIP_DOWNLOAD}" != true ] && [ "${INSTALL_K3S_SKIP_DOWNLOAD}" != selinux ]; then 
        return 1                                                                     
    fi                                                                               
}  

# --- verify an executable k3s binary is installed ---
verify_k3s_is_executable() {
    if [ ! -x ${BIN_DIR}/k3s ]; then
        fatal "Executable k3s binary not found at ${BIN_DIR}/k3s"
    fi
}

# --- set arch and suffix, fatal if architecture not supported ---
setup_verify_arch() {
    if [ -z "$ARCH" ]; then
        ARCH=$(uname -m)
    fi
    case $ARCH in
        amd64)
            ARCH=amd64
            SUFFIX=
            ;;
        x86_64)
            ARCH=amd64
            SUFFIX=
            ;;
        arm64)
            ARCH=arm64
            SUFFIX=-${ARCH}
            ;;
        s390x)
            ARCH=s390x
            SUFFIX=-${ARCH}
            ;;
        aarch64)
            ARCH=arm64
            SUFFIX=-${ARCH}
            ;;
        arm*)
            ARCH=arm
            SUFFIX=-${ARCH}hf
            ;;
        *)
            fatal "Unsupported architecture $ARCH"
    esac
}

# --- verify existence of network downloader executable ---
verify_downloader() {
    # Return failure if it doesn't exist or is no executable
    [ -x "$(command -v $1)" ] || return 1

    # Set verified executable as our downloader program and return success
    DOWNLOADER=$1
    return 0
}

# --- create temporary directory and cleanup when done ---
setup_tmp() {
    TMP_DIR=$(mktemp -d -t k3s-install.XXXXXXXXXX)
    TMP_HASH=${TMP_DIR}/k3s.hash
    TMP_BIN=${TMP_DIR}/k3s.bin
    cleanup() {
        code=$?
        set +e
        trap - EXIT
        rm -rf ${TMP_DIR}
        exit $code
    }
    trap cleanup INT EXIT
}

# --- use desired k3s version if defined or find version from channel ---
get_release_version() {
    if [ -n "${INSTALL_K3S_COMMIT}" ]; then
        VERSION_K3S="commit ${INSTALL_K3S_COMMIT}"
    elif [ -n "${INSTALL_K3S_VERSION}" ]; then
        VERSION_K3S=${INSTALL_K3S_VERSION}
    else
        info "Finding release for channel ${INSTALL_K3S_CHANNEL}"
        version_url="${INSTALL_K3S_CHANNEL_URL}/${INSTALL_K3S_CHANNEL}"
        case $DOWNLOADER in
            curl)
                if [ "${INSTALL_K3S_MIRROR}" = cn ]; then
                    VERSION_K3S=$(curl -s -S ${version_url})
                else
                    VERSION_K3S=$(curl -w '%{url_effective}' -L -s -S ${version_url} -o /dev/null | sed -e 's|.*/||')
                fi
                ;;
            wget)
                if [ "${INSTALL_K3S_MIRROR}" = cn ]; then
                    VERSION_K3S=$(wget -qO - ${version_url})
                else
                    VERSION_K3S=$(wget -SqO /dev/null ${version_url} 2>&1 | grep -i Location | sed -e 's|.*/||')
                fi
                ;;
            *)
                fatal "Incorrect downloader executable '$DOWNLOADER'"
                ;;
        esac
    fi
    info "Using ${VERSION_K3S} as release"
}

# --- download from github url ---
download() {
    [ $# -eq 2 ] || fatal 'download needs exactly 2 arguments'

    case $DOWNLOADER in
        curl)
            curl -o $1 -sfL $2
            ;;
        wget)
            wget -qO $1 $2
            ;;
        *)
            fatal "Incorrect executable '$DOWNLOADER'"
            ;;
    esac

    # Abort if download command failed
    [ $? -eq 0 ] || fatal 'Download failed'
}

# --- download hash from github url ---
download_hash() {
    if [ -n "${INSTALL_K3S_COMMIT}" ]; then
        HASH_URL=${STORAGE_URL}/k3s${SUFFIX}-${INSTALL_K3S_COMMIT}.sha256sum
    elif [ "${INSTALL_K3S_MIRROR}" = cn ]; then
        VERSION_K3S=$( echo ${VERSION_K3S} | sed 's/+/-/g' )
        HASH_URL=${INSTALL_K3S_MIRROR_URL}/k3s/${VERSION_K3S}/sha256sum-${ARCH}.txt
    else
        HASH_URL=${GITHUB_URL}/download/${VERSION_K3S}/sha256sum-${ARCH}.txt
    fi
    info "Downloading hash ${HASH_URL}"
    download ${TMP_HASH} ${HASH_URL}
    HASH_EXPECTED=$(grep " k3s${SUFFIX}$" ${TMP_HASH})
    HASH_EXPECTED=${HASH_EXPECTED%%[[:blank:]]*}
}

# --- check hash against installed version ---
installed_hash_matches() {
    if [ -x ${BIN_DIR}/k3s ]; then
        HASH_INSTALLED=$(sha256sum ${BIN_DIR}/k3s)
        HASH_INSTALLED=${HASH_INSTALLED%%[[:blank:]]*}
        if [ "${HASH_EXPECTED}" = "${HASH_INSTALLED}" ]; then
            return
        fi
    fi
    return 1
}

# --- download binary from github url ---
download_binary() {
    if [ -n "${INSTALL_K3S_COMMIT}" ]; then
        BIN_URL=${STORAGE_URL}/k3s${SUFFIX}-${INSTALL_K3S_COMMIT}
    elif [ "${INSTALL_K3S_MIRROR}" = cn ]; then
        VERSION_K3S=$( echo ${VERSION_K3S} | sed 's/+/-/g' )
        BIN_URL=${INSTALL_K3S_MIRROR_URL}/k3s/${VERSION_K3S}/k3s${SUFFIX}
    else
        BIN_URL=${GITHUB_URL}/download/${VERSION_K3S}/k3s${SUFFIX}
    fi
    info "Downloading binary ${BIN_URL}"
    download ${TMP_BIN} ${BIN_URL}
}

# --- verify downloaded binary hash ---
verify_binary() {
    info "Verifying binary download"
    HASH_BIN=$(sha256sum ${TMP_BIN})
    HASH_BIN=${HASH_BIN%%[[:blank:]]*}
    if [ "${HASH_EXPECTED}" != "${HASH_BIN}" ]; then
        fatal "Download sha256 does not match ${HASH_EXPECTED}, got ${HASH_BIN}"
    fi
}

# --- setup permissions and move binary to system directory ---
setup_binary() {
    chmod 755 ${TMP_BIN}
    info "Installing k3s to ${BIN_DIR}/k3s"
    $SUDO chown root:root ${TMP_BIN}
    $SUDO mv -f ${TMP_BIN} ${BIN_DIR}/k3s
}

# --- setup selinux policy ---
setup_selinux() {
    case ${INSTALL_K3S_CHANNEL} in 
        *testing)
            rpm_channel=testing
            ;;
        *latest)
            rpm_channel=latest
            ;;
        *)
            rpm_channel=stable
            ;;
    esac

    rpm_site="rpm.rancher.io"
    if [ "${rpm_channel}" = "testing" ]; then
        rpm_site="rpm-testing.rancher.io"
    fi

    [ -r /etc/os-release ] && . /etc/os-release
    if [ `expr "${ID_LIKE}" : ".*suse.*"` != 0 ]; then
        rpm_target=sle
        rpm_site_infix=microos
        package_installer=zypper
    elif [ "${VERSION_ID%%.*}" = "7" ]; then
        rpm_target=el7
        rpm_site_infix=centos/7
        package_installer=yum
    else
        rpm_target=el8
        rpm_site_infix=centos/8
        package_installer=yum
    fi

    if [ "${package_installer}" = "yum" ] && [ -x /usr/bin/dnf ]; then
        package_installer=dnf
    fi

    policy_hint="please install:
    ${package_installer} install -y container-selinux
    ${package_installer} install -y https://${rpm_site}/k3s/${rpm_channel}/common/${rpm_site_infix}/noarch/k3s-selinux-1.2-2.${rpm_target}.noarch.rpm
"

    if [ "$INSTALL_K3S_SKIP_SELINUX_RPM" = true ] || can_skip_download_selinux || [ ! -d /usr/share/selinux ]; then
        info "Skipping installation of SELinux RPM"
    elif  [ "${ID_LIKE:-}" != coreos ] && [ "${VARIANT_ID:-}" != coreos ]; then
        install_selinux_rpm ${rpm_site} ${rpm_channel} ${rpm_target} ${rpm_site_infix}
    fi

    policy_error=fatal
    if [ "$INSTALL_K3S_SELINUX_WARN" = true ] || [ "${ID_LIKE:-}" = coreos ] || [ "${VARIANT_ID:-}" = coreos ]; then
        policy_error=warn
    fi

    if ! $SUDO chcon -u system_u -r object_r -t container_runtime_exec_t ${BIN_DIR}/k3s >/dev/null 2>&1; then
        if $SUDO grep '^\s*SELINUX=enforcing' /etc/selinux/config >/dev/null 2>&1; then
            $policy_error "Failed to apply container_runtime_exec_t to ${BIN_DIR}/k3s, ${policy_hint}"
        fi
    elif [ ! -f /usr/share/selinux/packages/k3s.pp ]; then
        if [ -x /usr/sbin/transactional-update ]; then
            warn "Please reboot your machine to activate the changes and avoid data loss."
        else
            $policy_error "Failed to find the k3s-selinux policy, ${policy_hint}"
        fi
    fi
}

install_selinux_rpm() {
    if [ -r /etc/redhat-release ] || [ -r /etc/centos-release ] || [ -r /etc/oracle-release ] || [ "${ID_LIKE%%[ ]*}" = "suse" ]; then
        repodir=/etc/yum.repos.d
        if [ -d /etc/zypp/repos.d ]; then
            repodir=/etc/zypp/repos.d
        fi
        set +o noglob
        $SUDO rm -f ${repodir}/rancher-k3s-common*.repo
        set -o noglob
        if [ -r /etc/redhat-release ] && [ "${3}" = "el7" ]; then
            $SUDO yum install -y yum-utils
            $SUDO yum-config-manager --enable rhel-7-server-extras-rpms
        fi
        $SUDO tee ${repodir}/rancher-k3s-common.repo >/dev/null << EOF
[rancher-k3s-common-${2}]
name=Rancher K3s Common (${2})
baseurl=https://${1}/k3s/${2}/common/${4}/noarch
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=https://${1}/public.key
EOF
        case ${3} in
        sle)
            rpm_installer="zypper --gpg-auto-import-keys"
            if [ "${TRANSACTIONAL_UPDATE=false}" != "true" ] && [ -x /usr/sbin/transactional-update ]; then
                rpm_installer="transactional-update --no-selfupdate -d run ${rpm_installer}"
                : "${INSTALL_K3S_SKIP_START:=true}"
            fi
            ;;
        *)
            rpm_installer="yum"
            ;;
        esac
        if [ "${rpm_installer}" = "yum" ] && [ -x /usr/bin/dnf ]; then
            rpm_installer=dnf
        fi
        # shellcheck disable=SC2086
        $SUDO ${rpm_installer} install -y "k3s-selinux"
    fi
    return
}

# --- download and verify k3s ---
download_and_verify() {
    if can_skip_download_binary; then
       info 'Skipping k3s download and verify'
       verify_k3s_is_executable
       return
    fi

    setup_verify_arch
    verify_downloader curl || verify_downloader wget || fatal 'Can not find curl or wget for downloading files'
    setup_tmp
    get_release_version
    download_hash

    if installed_hash_matches; then
        info 'Skipping binary downloaded, installed k3s matches hash'
        return
    fi

    download_binary
    verify_binary
    setup_binary
}

# --- add additional utility links ---
create_symlinks() {
    [ "${INSTALL_K3S_BIN_DIR_READ_ONLY}" = true ] && return
    [ "${INSTALL_K3S_SYMLINK}" = skip ] && return

    for cmd in kubectl crictl ctr; do
        if [ ! -e ${BIN_DIR}/${cmd} ] || [ "${INSTALL_K3S_SYMLINK}" = force ]; then
            which_cmd=$(command -v ${cmd} 2>/dev/null || true)
            if [ -z "${which_cmd}" ] || [ "${INSTALL_K3S_SYMLINK}" = force ]; then
                info "Creating ${BIN_DIR}/${cmd} symlink to k3s"
                $SUDO ln -sf k3s ${BIN_DIR}/${cmd}
            else
                info "Skipping ${BIN_DIR}/${cmd} symlink to k3s, command exists in PATH at ${which_cmd}"
            fi
        else
            info "Skipping ${BIN_DIR}/${cmd} symlink to k3s, already exists"
        fi
    done
}

# --- create killall script ---
create_killall() {
    [ "${INSTALL_K3S_BIN_DIR_READ_ONLY}" = true ] && return
    info "Creating killall script ${KILLALL_K3S_SH}"
    $SUDO tee ${KILLALL_K3S_SH} >/dev/null << \EOF
#!/bin/sh
[ $(id -u) -eq 0 ] || exec sudo $0 $@

for bin in /var/lib/rancher/k3s/data/**/bin/; do
    [ -d $bin ] && export PATH=$PATH:$bin:$bin/aux
done

set -x

for service in /etc/systemd/system/k3s*.service; do
    [ -s $service ] && systemctl stop $(basename $service)
done

for service in /etc/init.d/k3s*; do
    [ -x $service ] && $service stop
done

pschildren() {
    ps -e -o ppid= -o pid= | \
    sed -e 's/^\s*//g; s/\s\s*/\t/g;' | \
    grep -w "^$1" | \
    cut -f2
}

pstree() {
    for pid in $@; do
        echo $pid
        for child in $(pschildren $pid); do
            pstree $child
        done
    done
}

killtree() {
    kill -9 $(
        { set +x; } 2>/dev/null;
        pstree $@;
        set -x;
    ) 2>/dev/null
}

getshims() {
    ps -e -o pid= -o args= | sed -e 's/^ *//; s/\s\s*/\t/;' | grep -w 'k3s/data/[^/]*/bin/containerd-shim' | cut -f1
}

killtree $({ set +x; } 2>/dev/null; getshims; set -x)

do_unmount_and_remove() {
    set +x
    while read -r _ path _; do
        case "$path" in $1*) echo "$path" ;; esac
    done < /proc/self/mounts | sort -r | xargs -r -t -n 1 sh -c 'umount "$0" && rm -rf "$0"'
    set -x
}

do_unmount_and_remove '/run/k3s'
do_unmount_and_remove '/var/lib/rancher/k3s'
do_unmount_and_remove '/var/lib/kubelet/pods'
do_unmount_and_remove '/var/lib/kubelet/plugins'
do_unmount_and_remove '/run/netns/cni-'

# Remove CNI namespaces
ip netns show 2>/dev/null | grep cni- | xargs -r -t -n 1 ip netns delete

# Delete network interface(s) that match 'master cni0'
ip link show 2>/dev/null | grep 'master cni0' | while read ignore iface ignore; do
    iface=${iface%%@*}
    [ -z "$iface" ] || ip link delete $iface
done
ip link delete cni0
ip link delete flannel.1
ip link delete flannel-v6.1
ip link delete kube-ipvs0
ip link delete flannel-wg
ip link delete flannel-wg-v6
rm -rf /var/lib/cni/
iptables-save | grep -v KUBE- | grep -v CNI- | grep -iv flannel | iptables-restore
ip6tables-save | grep -v KUBE- | grep -v CNI- | grep -iv flannel | ip6tables-restore
EOF
    $SUDO chmod 755 ${KILLALL_K3S_SH}
    $SUDO chown root:root ${KILLALL_K3S_SH}
}

# --- create uninstall script ---
create_uninstall() {
    [ "${INSTALL_K3S_BIN_DIR_READ_ONLY}" = true ] && return
    info "Creating uninstall script ${UNINSTALL_K3S_SH}"
    $SUDO tee ${UNINSTALL_K3S_SH} >/dev/null << EOF
#!/bin/sh
set -x
[ \$(id -u) -eq 0 ] || exec sudo \$0 \$@

${KILLALL_K3S_SH}

if command -v systemctl; then
    systemctl disable ${SYSTEM_NAME}
    systemctl reset-failed ${SYSTEM_NAME}
    systemctl daemon-reload
fi
if command -v rc-update; then
    rc-update delete ${SYSTEM_NAME} default
fi

rm -f ${FILE_K3S_SERVICE}
rm -f ${FILE_K3S_ENV}

remove_uninstall() {
    rm -f ${UNINSTALL_K3S_SH}
}
trap remove_uninstall EXIT

if (ls ${SYSTEMD_DIR}/k3s*.service || ls /etc/init.d/k3s*) >/dev/null 2>&1; then
    set +x; echo 'Additional k3s services installed, skipping uninstall of k3s'; set -x
    exit
fi

for cmd in kubectl crictl ctr; do
    if [ -L ${BIN_DIR}/\$cmd ]; then
        rm -f ${BIN_DIR}/\$cmd
    fi
done

rm -rf /etc/rancher/k3s
rm -rf /run/k3s
rm -rf /run/flannel
rm -rf /var/lib/rancher/k3s
rm -rf /var/lib/kubelet
rm -f ${BIN_DIR}/k3s
rm -f ${KILLALL_K3S_SH}

if type yum >/dev/null 2>&1; then
    yum remove -y k3s-selinux
    rm -f /etc/yum.repos.d/rancher-k3s-common*.repo
elif type zypper >/dev/null 2>&1; then
    uninstall_cmd="zypper remove -y k3s-selinux"
    if [ "\${TRANSACTIONAL_UPDATE=false}" != "true" ] && [ -x /usr/sbin/transactional-update ]; then
        uninstall_cmd="transactional-update --no-selfupdate -d run \$uninstall_cmd"
    fi
    \$uninstall_cmd
    rm -f /etc/zypp/repos.d/rancher-k3s-common*.repo
fi
EOF
    $SUDO chmod 755 ${UNINSTALL_K3S_SH}
    $SUDO chown root:root ${UNINSTALL_K3S_SH}
}

# --- disable current service if loaded --
systemd_disable() {
    $SUDO systemctl disable ${SYSTEM_NAME} >/dev/null 2>&1 || true
    $SUDO rm -f /etc/systemd/system/${SERVICE_K3S} || true
    $SUDO rm -f /etc/systemd/system/${SERVICE_K3S}.env || true
}

# --- capture current env and create file containing k3s_ variables ---
create_env_file() {
    info "env: Creating environment file ${FILE_K3S_ENV}"
    $SUDO touch ${FILE_K3S_ENV}
    $SUDO chmod 0600 ${FILE_K3S_ENV}
    sh -c export | while read x v; do echo $v; done | grep -E '^(K3S|CONTAINERD)_' | $SUDO tee ${FILE_K3S_ENV} >/dev/null
    sh -c export | while read x v; do echo $v; done | grep -Ei '^(NO|HTTP|HTTPS)_PROXY' | $SUDO tee -a ${FILE_K3S_ENV} >/dev/null
}

# --- write systemd service file ---
create_systemd_service_file() {
    info "systemd: Creating service file ${FILE_K3S_SERVICE}"
    $SUDO tee ${FILE_K3S_SERVICE} >/dev/null << EOF
[Unit]
Description=Lightweight Kubernetes
Documentation=https://k3s.io
Wants=network-online.target
After=network-online.target

[Install]
WantedBy=multi-user.target

[Service]
Type=${SYSTEMD_TYPE}
EnvironmentFile=-/etc/default/%N
EnvironmentFile=-/etc/sysconfig/%N
EnvironmentFile=-${FILE_K3S_ENV}
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
ExecStart=${BIN_DIR}/k3s \\
    ${CMD_K3S_EXEC}

EOF
}

# --- write openrc service file ---
create_openrc_service_file() {
    LOG_FILE=/var/log/${SYSTEM_NAME}.log

    info "openrc: Creating service file ${FILE_K3S_SERVICE}"
    $SUDO tee ${FILE_K3S_SERVICE} >/dev/null << EOF
#!/sbin/openrc-run

depend() {
    after network-online
    want cgroups
}

start_pre() {
    rm -f /tmp/k3s.*
}

supervisor=supervise-daemon
name=${SYSTEM_NAME}
command="${BIN_DIR}/k3s"
command_args="$(escape_dq "${CMD_K3S_EXEC}")
    >>${LOG_FILE} 2>&1"

output_log=${LOG_FILE}
error_log=${LOG_FILE}

pidfile="/var/run/${SYSTEM_NAME}.pid"
respawn_delay=5
respawn_max=0

set -o allexport
if [ -f /etc/environment ]; then sourcex /etc/environment; fi
if [ -f ${FILE_K3S_ENV} ]; then sourcex ${FILE_K3S_ENV}; fi
set +o allexport
EOF
    $SUDO chmod 0755 ${FILE_K3S_SERVICE}

    $SUDO tee /etc/logrotate.d/${SYSTEM_NAME} >/dev/null << EOF
${LOG_FILE} {
	missingok
	notifempty
	copytruncate
}
EOF
}

# --- write systemd or openrc service file ---
create_service_file() {
    [ "${HAS_SYSTEMD}" = true ] && create_systemd_service_file
    [ "${HAS_OPENRC}" = true ] && create_openrc_service_file
    return 0
}

# --- get hashes of the current k3s bin and service files
get_installed_hashes() {
    $SUDO sha256sum ${BIN_DIR}/k3s ${FILE_K3S_SERVICE} ${FILE_K3S_ENV} 2>&1 || true
}

# --- enable and start systemd service ---
systemd_enable() {
    info "systemd: Enabling ${SYSTEM_NAME} unit"
    $SUDO systemctl enable ${FILE_K3S_SERVICE} >/dev/null
    $SUDO systemctl daemon-reload >/dev/null
}

systemd_start() {
    info "systemd: Starting ${SYSTEM_NAME}"
    $SUDO systemctl restart ${SYSTEM_NAME}
}

# --- enable and start openrc service ---
openrc_enable() {
    info "openrc: Enabling ${SYSTEM_NAME} service for default runlevel"
    $SUDO rc-update add ${SYSTEM_NAME} default >/dev/null
}

openrc_start() {
    info "openrc: Starting ${SYSTEM_NAME}"
    $SUDO ${FILE_K3S_SERVICE} restart
}

# --- startup systemd or openrc service ---
service_enable_and_start() {
    if [ -f "/proc/cgroups" ] && [ "$(grep memory /proc/cgroups | while read -r n n n enabled; do echo $enabled; done)" -eq 0 ];
    then
        info 'Failed to find memory cgroup, you may need to add "cgroup_memory=1 cgroup_enable=memory" to your linux cmdline (/boot/cmdline.txt on a Raspberry Pi)'
    fi

    [ "${INSTALL_K3S_SKIP_ENABLE}" = true ] && return

    [ "${HAS_SYSTEMD}" = true ] && systemd_enable
    [ "${HAS_OPENRC}" = true ] && openrc_enable

    [ "${INSTALL_K3S_SKIP_START}" = true ] && return

    POST_INSTALL_HASHES=$(get_installed_hashes)
    if [ "${PRE_INSTALL_HASHES}" = "${POST_INSTALL_HASHES}" ] && [ "${INSTALL_K3S_FORCE_RESTART}" != true ]; then
        info 'No change detected so skipping service start'
        return
    fi

    [ "${HAS_SYSTEMD}" = true ] && systemd_start
    [ "${HAS_OPENRC}" = true ] && openrc_start
    return 0
}

# --- re-evaluate args to include env command ---
eval set -- $(escape "${INSTALL_K3S_EXEC}") $(quote "$@")

# --- run the install process --
{
    verify_system
    setup_env "$@"
    download_and_verify
    setup_selinux
    create_symlinks
    create_killall
    create_uninstall
    systemd_disable
    create_env_file
    create_service_file
    service_enable_and_start
}
{
"type": "collection",
"links": {
"self": "…/v1-release/channels"
},
"actions": { },
"resourceType": "channels",
"data": [ 17 items
{
"id": "stable",
"type": "channel",
"links": {
"self": "…/v1-release/channels/stable"
},
"name": "stable",
"latest": "v1.25.6+k3s1"
},
{
"id": "latest",
"type": "channel",
"links": {
"self": "…/v1-release/channels/latest"
},
"name": "latest",
"latest": "v1.26.1+k3s1",
"latestRegexp": ".*",
"excludeRegexp": "(^[^+]+-|v1\\.25\\.5\\+k3s1|v1\\.26\\.0\\+k3s1)"
},
{
"id": "testing",
"type": "channel",
"links": {
"self": "…/v1-release/channels/testing"
},
"name": "testing",
"latest": "v1.26.2-rc1+k3s1",
"latestRegexp": "-(alpha|beta|rc)"
},
{
"id": "v1.16",
"type": "channel",
"links": {
"self": "…/v1-release/channels/v1.16"
},
"name": "v1.16",
"latest": "v1.16.15+k3s1",
"latestRegexp": "v1\\.16\\..*",
"excludeRegexp": "^[^+]+-"
},
{
"id": "v1.16-testing",
"type": "channel",
"links": {
"self": "…/v1-release/channels/v1.16-testing"
},
"name": "v1.16-testing",
"latest": "v1.16.14-rc1+k3s1",
"latestRegexp": "v1\\.16\\.[0-9]*-(alpha|beta|rc)"
},
{
"id": "v1.17",
"type": "channel",
"links": {
"self": "…/v1-release/channels/v1.17"
},
"name": "v1.17",
"latest": "v1.17.17+k3s1",
"latestRegexp": "v1\\.17\\..*",
"excludeRegexp": "^[^+]+-"
},
{
"id": "v1.17-testing",
"type": "channel",
"links": {
"self": "…/v1-release/channels/v1.17-testing"
},
"name": "v1.17-testing",
"latest": "v1.17.17-rc1+k3s1",
"latestRegexp": "v1\\.17\\.[0-9]*-(alpha|beta|rc)"
},
{
"id": "v1.18",
"type": "channel",
"links": {
"self": "…/v1-release/channels/v1.18"
},
"name": "v1.18",
"latest": "v1.18.20+k3s1",
"latestRegexp": "v1\\.18\\..*",
"excludeRegexp": "^[^+]+-"
},
{
"id": "v1.18-testing",
"type": "channel",
"links": {
"self": "…/v1-release/channels/v1.18-testing"
},
"name": "v1.18-testing",
"latest": "v1.18.20-rc1+k3s1",
"latestRegexp": "v1\\.18\\.[0-9]*-(alpha|beta|rc)"
},
{
"id": "v1.19",
"type": "channel",
"links": {
"self": "…/v1-release/channels/v1.19"
},
"name": "v1.19",
"latest": "v1.19.16+k3s1",
"latestRegexp": "v1\\.19\\..*",
"excludeRegexp": "^[^+]+-"
},
{
"id": "v1.20",
"type": "channel",
"links": {
"self": "…/v1-release/channels/v1.20"
},
"name": "v1.20",
"latest": "v1.20.15+k3s1",
"latestRegexp": "v1\\.20\\..*",
"excludeRegexp": "^[^+]+-"
},
{
"id": "v1.21",
"type": "channel",
"links": {
"self": "…/v1-release/channels/v1.21"
},
"name": "v1.21",
"latest": "v1.21.14+k3s1",
"latestRegexp": "v1\\.21\\..*",
"excludeRegexp": "^[^+]+-"
},
{
"id": "v1.22",
"type": "channel",
"links": {
"self": "…/v1-release/channels/v1.22"
},
"name": "v1.22",
"latest": "v1.22.17+k3s1",
"latestRegexp": "v1\\.22\\..*",
"excludeRegexp": "^[^+]+-"
},
{
"id": "v1.23",
"type": "channel",
"links": {
"self": "…/v1-release/channels/v1.23"
},
"name": "v1.23",
"latest": "v1.23.16+k3s1",
"latestRegexp": "v1\\.23\\..*",
"excludeRegexp": "^[^+]+-"
},
{
"id": "v1.24",
"type": "channel",
"links": {
"self": "…/v1-release/channels/v1.24"
},
"name": "v1.24",
"latest": "v1.24.10+k3s1",
"latestRegexp": "v1\\.24\\..*",
"excludeRegexp": "(^[^+]+-|v1\\.24\\.9\\+k3s1)"
},
{
"id": "v1.25",
"type": "channel",
"links": {
"self": "…/v1-release/channels/v1.25"
},
"name": "v1.25",
"latest": "v1.25.6+k3s1",
"latestRegexp": "v1\\.25\\..*",
"excludeRegexp": "(^[^+]+-|v1\\.25\\.5\\+k3s1)"
},
{
"id": "v1.26",
"type": "channel",
"links": {
"self": "…/v1-release/channels/v1.26"
},
"name": "v1.26",
"latest": "v1.26.1+k3s1",
"latestRegexp": "v1\\.26\\..*",
"excludeRegexp": "(^[^+]+-|v1\\.26\\.0\\+k3s1)"
}
]
}
```