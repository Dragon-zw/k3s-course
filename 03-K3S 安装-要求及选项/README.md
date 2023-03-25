Video Reference：[03-安装-要求及选项](https://www.bilibili.com/video/BV1Gu411m7vx/?spm_id_from=333.788&vd_source=9560c118fae1db9638f05a6ba2527085)

GitHub README.md：[03-安装-要求及选项](https://github.com/kingsd041/k3s-tutorial/blob/main/03-安装-要求及选项/README.md)

## 1 安装要求

### 1.1 先决条件

- 两个节点不能有相同的主机名。
- 不修改主机名可以通过添加随机后缀或指定主机名

如果 k3s master 和 k3s worker 节点主机名相同，注册 worker 节点时会报错：

```properties
# 造成这种错误的原因：
# 报错解释：集群中拥有重复的Hostname
# 可能涉及到 K3s agent 的注册流程，首先如果想把K3s agent注册到K3s master中
# K3s agent本地首先会生成一串随机Password，K3s agent结合Password跟主机名Hostname传到K3s master
# 在老版本的K3s，password 是存储在/etc/rancher目录中
Apr 13 07:37:37 k3s2 k3s[1763]: time="2021-04-13T15:37:37.444123090+08:00" level=error msg="Failed to retrieve agent config: Node password rejected, duplicate hostname or contents of '/etc/rancher/node/password' may not match server node-passwd entry, try enabling a unique node name with the --with-node-id flag"
```

-  如果您的所有节点都有相同的主机名，请使用`--with-node-id`选项为每个节点添加一个随机后缀，或者为您添加到集群的每个节点设计一个独特的名称，用`--node-name`或`$K3S_NODE_NAME`传递。 

💡Tips

在没有明确设置 exec 命令的情况下设置`K3S_URL`，会将命令默认为 `"agent"`。

运行 agent 时还必须设置`K3S_TOKEN`。

- - `--with-node-id`：

```bash
# 启动一个K3s Master节点
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN="rancher" \
  sh

# 为每个节点添加随机后缀
# K3S_URL : 指定K3s URL
# K3S_TOKEN : 指定K3s Master Token[cat /var/lib/rancher/k3s/server/token]
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_URL=https://10.0.0.30:6443 \
  K3S_TOKEN=rancher \
  sh -s - --with-node-id
$ kubectl get nodes
NAME            STATUS   ROLES                  AGE   VERSION
k3s1            Ready    control-plane,master   67m   v1.21.14+k3s1
k3s2-3e0439f4   Ready    <none>                 15s   v1.21.14+k3s1
```

- - `指定主机名`：

```bash
# 为每个节点指定主机名
# K3S_URL : 指定K3s Master URL
# K3S_TOKEN : 指定K3s Master Token[cat /var/lib/rancher/k3s/server/token]
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  K3S_URL=https://10.0.0.30:6443 \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN=rancher sh -s - --node-name k3s2
# 为每个节点指定主机名
# K3S_URL : 指定K3s Master URL
# K3S_TOKEN : 指定K3s Master Token[cat /var/lib/rancher/k3s/server/token]
# 使用环境变量的方式指定主机名最后还是会以参数的形式进行修改
$ curl -sfL https://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | \
  K3S_NODE_NAME="k3s2" \
  INSTALL_K3S_MIRROR=cn \
  K3S_URL=https://10.0.0.30:6443 \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN=rancher sh -
$ kubectl get nodes
NAME   STATUS   ROLES                  AGE   VERSION
k3s1   Ready    control-plane,master   72m   v1.21.14+k3s1
k3s2   Ready    <none>                 28s   v1.21.14+k3s1
```

### 1.2 操作系统

K3s 有望在大多数现代 Linux 系统上运行:

- CentOS: 7.8, 7.9, 8.2, 8.3
- RHEL: 7.8, 7.9, 8.2, 8.3
- SLES: 15SP2
- Ubuntu: 16.04, 18.04, 20.04
- 国内的 Linux 操作系统，例如麒麟系统

### 1.3 硬件

#### 1.3.1 CPU & 内存

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

#### 1.3.2 网络

**K3s Server 节点的入站规则：**

| **协议** | **端口**  | **源**                   | **描述**                                                   |
| -------- | --------- | ------------------------ | ---------------------------------------------------------- |
| TCP      | 6443      | K3s agent 节点           | Kubernetes API Server                                      |
| UDP      | 8472      | K3s server 和 agent 节点 | 仅对 Flannel VXLAN 需要                                    |
| TCP      | 10250     | K3s server 和 agent 节点 | Kubelet metrics                                            |
| TCP      | 2379-2380 | K3s server 节点          | 只有嵌入式 etcd 高可用才需要，使用SQLite数据库则不需要开启 |

**通常情况下，所有出站流量都是允许的。**

## 2 K3s 资源分析

### 2.1 影响资源利用率的因素

K3s server 的利用率数据主要是由支持 Kubernetes 数据存储（kine 或 etcd）、API Server、Controller-Manager 和 Scheduler 控制，以及实现系统状态变化所需的任何管理任务所驱动。给 Kubernetes 控制平面带来额外负载的操作，如创建/修改/删除资源，将导致暂时的利用率上升。使用大量使用 Kubernetes 数据存储的 operators 或应用程序（如 Rancher 或其他 operators 类型的应用程序）将增加 server 的资源需求。通过添加额外的节点或创建许多集群资源来扩展集群，将增加 server 的资源需求。

K3s agent 的利用率数据主要是由支持容器生命周期管理控制驱动的。涉及管理镜像、提供存储或创建/销毁容器/解压镜像文件的操作将导致利用率的暂时上升。拉取镜像通常会影响 CPU 和 IO，因为它们涉及将镜像内容解压到磁盘。如果可能的话，工作负载存储`(pod 临时存储和卷)`应该与 agent 组件(`/var/lib/rancher/k3s/agent`)隔离，以确保不会出现资源冲突。

### 2.2 防止 agent 和工作负载干扰集群数据存储

在 server 节点运行工作负载 pod 时，应确保 agent 和工作负载 IOPS 不干扰数据存储。

将 server 组件（`/var/lib/rancher/k3s/server`）与 agent 组件（`/var/lib/rancher/k3s/agent`）放在不同的存储介质上，后者包括 containerd 镜像存储。

工作负载存储（pod 临时存储和卷）也应该与数据存储隔离。

如果不能满足数据存储的吞吐量和延迟要求，可能会导致控制平面的延迟响应或控制平面无法维持系统状态。

## 3 安装选项

### 3.1 使用脚本安装的选项

- [官方安装参数文档](https://docs.rancher.cn/docs/k3s/autok3s/_index)
- [安装选项示例演示](https://github.com/kingsd041/k3s-tutorial/blob/main/03-安装-要求及选项/README.md)

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
| `K3S_CONFIG_FILE`               | 指定配置文件的位置。默认目录为`/etc/rancher/k3s/config.yaml`。 |
| `K3S_TOKEN`                     | 用于将 server 或 agent 加入集群的共享 secret。               |
| `K3S_TOKEN_FILE`                | 指定 cluster-secret,token 的文件目录。                       |

#### 3.1.1 默认的启动过程

- 默认的启动过程

```shell
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn sh -
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
# 可以通过systemd的服务的方式启动和关闭k3s服务
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
# 启动K3s集群
[INFO]  systemd: Enabling k3s unit
Created symlink from /etc/systemd/system/multi-user.target.wants/k3s.service to /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s
```

#### 3.1.2 INSTALL_K3S_SKIP_DOWNLOAD 

- `INSTALL_K3S_SKIP_DOWNLOAD` -- (用于离线安装) 如果设置为 "true "将不会下载 K3s 的哈希值或二进制。 该环境变量使用较少。

```bash
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
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
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
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

#### 3.1.3 INSTALL_K3S_SYMLINK

- `INSTALL_K3S_SYMLINK` -- 默认情况下，如果路径中不存在命令，将为 kubectl、crictl 和 ctr 二进制文件创建符号链接。如果设置为'`skip`'将不会创建符号链接，而'`force`'将覆盖。该环境变量使用较少。

```bash
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
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

#### 3.1.4 INSTALL_K3S_SKIP_ENABLE

-  `INSTALL_K3S_SKIP_ENABLE` -- 如果设置为 "`true`"，将不启用或启动 K3s 服务。该环境变量使用较少。

```bash
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
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

# 查看k3s服务的状态
$ systemctl status k3s.service | grep "Active"
   Active: inactive (dead)

# 启动k3s服务
$ systemctl enable --now /etc/systemd/system/k3s.service
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
```

#### 3.1.5 INSTALL_K3S_SKIP_START

-  `INSTALL_K3S_SKIP_START` -- 如果设置为 "`true` "将不会启动 K3s 服务。该环境变量使用较少。

```bash
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
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

#### 3.1.6 👑INSTALL_K3S_VERSION

-  `INSTALL_K3S_VERSION` -- 从 Github 下载 K3s 的版本。如果没有指定，将尝试从"`stable`"频道下载。 

```bash
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
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

Reference：https://github.com/k3s-io/k3s/releases

#### 3.1.7 INSTALL_K3S_BIN_DIR

-  `INSTALL_K3S_BIN_DIR` -- 安装 K3s 二进制文件、链接和卸载脚本的目录，或者使用`/usr/local/bin`作为默认目录。该环境变量使用较少。

```bash
# 管理员若需要自定义二进制目录的路径，可以使用INSTALL_K3S_BIN_DIR环境变量
$ mkdir -pv /opt/bin/
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
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

#### 3.1.8 INSTALL_K3S_BIN_DIR_READ_ONLY

-  `INSTALL_K3S_BIN_DIR_READ_ONLY` -- 如果设置为 true 将不会把文件写入`INSTALL_K3S_BIN_DIR`，强制设置`INSTALL_K3S_SKIP_DOWNLOAD=true`。`INSTALL_K3S_SKIP_DOWNLOAD` 会创建`kubectl/crictl/ctr`等，而 `INSTALL_K3S_BIN_DIR_READ_ONLY` 不创建`kubectl/crictl/ctr`等。该环境变量使用较少。

```bash
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_BIN_DIR_READ_ONLY=true \
  sh -
[INFO]  Skipping k3s download and verify
[ERROR]  Executable k3s binary not found at /usr/local/bin/k3s

$ cp k3s /usr/local/bin/
$ curl -sfL hhttps://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
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

#### 3.1.9 INSTALL_K3S_SYSTEMD_DIR

-  `INSTALL_K3S_SYSTEMD_DIR` -- 安装 systemd 服务和环境文件的目录，或者使用`/etc/systemd/system` 作为默认目录。该环境变量使用较少。

```bash
$ mkdir -pv /opt/systemd
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
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

#### 3.1.10 👑INSTALL_K3S_EXEC

-  `INSTALL_K3S_EXEC` -- 带有标志的命令，用于在服务中启动 K3s。如果未指定命令，并且设置了 `K3S_URL`，它将默认为“`agent`”。如果未设置 `K3S_URL`，它将默认为“`server`”。 
- `INSTALL_K3S_EXEC` 后面追加的参数是在[K3s Server 配置参考](https://docs.rancher.cn/docs/k3s/installation/install-options/server-config/_index) 和 [K3s Agent配置参考](https://docs.rancher.cn/docs/k3s/installation/install-options/agent-config/_index)

```bash
# 其实就把对应参数加到systemd配置文件里面去了
# 需要有Docker的容器环境，才能使用 Docker 作为容器运行时环境
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
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

# 可以查看systemd的配置
$ cat /etc/systemd/system/k3s.service | grep -A 3 "\<ExecStart\>"
ExecStart=/usr/local/bin/k3s \
    server \
        '--docker' \
```

#### 3.1.11 INSTALL_K3S_NAME

-  `INSTALL_K3S_NAME` -- 要创建的 systemd 服务名称，如果以服务器方式运行 k3s，则默认为'k3s'；如果以 agent 方式运行 k3s，则默认为'k3s-agent'。如果指定了服务名，则`服务名将以'k3s-'为前缀`。 该环境变量使用较少。

```bash
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
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

#### 3.1.12 INSTALL_K3S_TYPE

-  `INSTALL_K3S_TYPE` -- 要创建的 systemd 服务类型，默认为 notify。该环境变量使用较少。

```bash
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
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

$ cat /etc/systemd/system/k3s.service | grep "Type"
Type=exec
```

#### 3.1.13 INSTALL_K3S_SKIP_SELINUX_RPM

-  `INSTALL_K3S_SKIP_SELINUX_RPM` -- 如果设置为 "`true` "将跳过 k3s RPM 的自动安装。 该环境变量使用较少。

```bash
# 即使是跳过了k3s-selinux的RPM包的安装
# 也是需要管理员手动进行k3s-selinux的RPM安装的
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
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

#### 3.1.14 INSTALL_K3S_CHANNEL_URL & INSTALL_K3S_CHANNEL

-  `INSTALL_K3S_CHANNEL_URL` -- 用于获取 K3s 下载网址的频道 URL。默认为 https://update.k3s.io/v1-release/channels 。已经实现了 CHANNEL的国产化，该环境变量使用较少。
-  `INSTALL_K3S_CHANNEL` -- 用于获取 K3s 下载 URL 的通道。默认值为 "`stable`"。选项包括：`stable`, `latest`, `testing`。 该环境变量使用较少。

```bash
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
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

#### 3.1.15 👑K3S_CONFIG_FILE

-  `K3S_CONFIG_FILE` -- 指定配置文件的位置。默认目录为`/etc/rancher/k3s/config.yaml`。

```bash
$ cat <<-'EOF' > /opt/config.yaml
node-label:
- "foo=bar"
- "something=amazing"
EOF

$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
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

#### 3.1.16 👑K3S_TOKEN

-  `K3S_TOKEN` -- 用于将 `server` 或 `agent` 加入集群的共享 `secret`。 在自动化环境中使用较多。

```bash
# 自动化部署(不用获取token值了)
# 主节点和工作节点使用我们指定的key来通信
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
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

#### 3.1.17 K3S_TOKEN_FILE

-  `K3S_TOKEN_FILE` -- 指定 `cluster-secret`,`token` 的文件目录。 和`K3S_TOKEN`大同小异，效果是一样的。该环境变量使用较少。

```bash
$ cat /opt/token.txt
rancher-k3s

$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
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

#### 3.1.18 其他说明

1. 以 "`K3S_`"开头的环境变量将被保留，供 `systemd` 和 `openrc` 服务使用。所以管理员在使用自定义环境变量的时候，尽量不要使用以 "`K3S_`"开头的环境变量
2. 在没有明确设置 `exec` 命令的情况下设置 `K3S_URL`，会将命令默认为 "`agent`"，即将该节点设置为"`K3s Agent`"的角色。
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
wget https://github.com/k3s-io/k3s/releases/download/v1.26.1%2Bk3s1/k3s -P /usr/local/bin/
```

#### 3.2.1 k3s server

-  `k3s server` -- 运行 K3s server，它还将启动 Kubernetes control-plane 组件，如 API server, controller-manager, 和 scheduler。 

```bash
$ chmod +x /usr/local/bin/k3s
# 需要占用控制台的进程
$ k3s server
INFO[0000] Acquiring lock file /var/lib/rancher/k3s/data/.lock
INFO[0000] Preparing data dir /var/lib/rancher/k3s/data/640b9915ffbb627ea4a2213ce03a1861e755a23967f45ab7241c094e10a6749f
INFO[0000] Starting k3s v1.26.1+k3s1 (f10af367)
[......]

# 默认是无法使用kubectl命令的，可以从Github中下载kubectl二进制文件
$ kubectl
bash: kubectl: command not found...

$ k3s kubectl get nodes
NAME   STATUS   ROLES                  AGE   VERSION
k3s1   Ready    control-plane,master   72s   v1.26.1+k3s1

# 查看k3s server的Token
$ cat /var/lib/rancher/k3s/server/token
K100b5b19986cf094d12e4c44e6124b07bcc90011c69e3641115fed952ba51127c4::server:d7b71cb80058833e51b892705876c19b
```

#### 3.2.2 k3s agent

-  `k3s agent` -- 运行 K3s agent 节点。这将使 K3s 作为工作节点运行，启动 Kubernetes 节点服务 kubelet 和 kube-proxy。 

```bash
# 需要占用控制台的进程
$ k3s agent --server https://10.0.0.51:6443 --token d7b71cb80058833e51b892705876c19b
INFO[0000] Acquiring lock file /var/lib/rancher/k3s/data/.lock
INFO[0000] Preparing data dir /var/lib/rancher/k3s/data/926943070e893920b703e893777d0cdc577dea7609f819abb14a97ea8ea1d30b
INFO[2021-04-15T10:09:57.962476104+08:00] Starting k3s agent v1.20.5+k3s1 (355fff30)
INFO[2021-04-15T10:09:58.351011343+08:00] Running load balancer 127.0.0.1:35105 -[192.168.64.3:6443]
[......]
```

#### 3.2.3 k3s kubectl

-  `k3s kubectl` -- 运行嵌入式 kubectl CLI。如果没有设置 KUBECONFIG 环境变量，当启动 K3s 服务器节点时，将自动尝试使用在`/etc/rancher/k3s/k3s.yaml` 创建的配置文件。 

```bash
$ k3s kubectl get nodes
NAME   STATUS   ROLES                  AGE   VERSION
k3s2   Ready    <none>                 8s    v1.26.1+k3s1
k3s1   Ready    control-plane,master   96s   v1.26.1+k3s1
```

#### 3.2.4 k3s crictl

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

#### 3.2.5 k3s ctr

-  `k3s ctr` -- 运行一个嵌入式的 ctr。这是为 containerd（K3s 使用的容器守护进程）提供的 CLI。对调试很有用。 

```bash
$ k3s ctr container ls
CONTAINER                                                           IMAGE                                                                                                               RUNTIME
01894072f2298208f3c109f9fb1d5e12e677d11cd5d0b0a3a66f550ae38644e4    docker.io/rancher/local-path-provisioner:v0.0.19                                                                    io.containerd.runc.v2
2bae007d8e486afffbbf1ffb88e97b92d367aff4b06842217de4fb5d22ecf1b9    docker.io/rancher/pause:3.1
```

`k3s server` 和 `k3s agent` 命令有额外的配置选项，可以通过 `k3s server --help` 或 `k3s agent --help` 查看.

### 3.3 通过配置文件启动 K3s

除了使用环境变量和 CLI 参数来配置 K3s，K3s 还可以使用配置文件，适合修改K3s 参数特别多的情况。默认目录位于`/etc/rancher/k3s/config.yaml`

```bash
$ mkdir -pv /etc/rancher/k3s
$ cat <<-'EOF' > /etc/rancher/k3s/config.yaml
node-label:
- "k3s-role=master"
EOF

# 需要占用控制台的进程
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

## 3.4 参考资料

K3s 脚本和版本迭代资料：

- https://rancher-mirror.rancher.cn/k3s/k3s-install.sh
- https://update.k3s.io/v1-release/channels

K3s 性能分析资料：

- [资源测试的范围](https://docs.rancher.cn/docs/k3s/installation/installation-requirements/resource-profiling/_index/#资源测试的范围)
- [基准测试包含的组件](https://docs.rancher.cn/docs/k3s/installation/installation-requirements/resource-profiling/_index/#基准测试包含的组件)
- [方法](https://docs.rancher.cn/docs/k3s/installation/installation-requirements/resource-profiling/_index/#方法)
- [环境](https://docs.rancher.cn/docs/k3s/installation/installation-requirements/resource-profiling/_index/#环境)
- [基准资源需求](https://docs.rancher.cn/docs/k3s/installation/installation-requirements/resource-profiling/_index/#基准资源需求)

- - [具有工作负载的 K3s server](https://docs.rancher.cn/docs/k3s/installation/installation-requirements/resource-profiling/_index/#具有工作负载的-k3s-server)
  - [具有单个 agent 的 K3s 集群](https://docs.rancher.cn/docs/k3s/installation/installation-requirements/resource-profiling/_index/#具有单个-agent-的-k3s-集群)
  - [K3s Agent](https://docs.rancher.cn/docs/k3s/installation/installation-requirements/resource-profiling/_index/#k3s-agent)

- [分析](https://docs.rancher.cn/docs/k3s/installation/installation-requirements/resource-profiling/_index/#分析)

- - [影响资源利用率的因素](https://docs.rancher.cn/docs/k3s/installation/installation-requirements/resource-profiling/_index/#影响资源利用率的因素)
  - [防止 agent 和工作负载干扰集群数据存储](https://docs.rancher.cn/docs/k3s/installation/installation-requirements/resource-profiling/_index/#防止-agent-和工作负载干扰集群数据存储)