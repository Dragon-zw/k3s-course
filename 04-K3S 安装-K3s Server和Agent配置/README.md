Video Reference：[04-安装-K3s Server和Agent配置](https://www.bilibili.com/video/BV1du41127YC/?spm_id_from=pageDriver&vd_source=9560c118fae1db9638f05a6ba2527085)

GitHub Reference：[04-安装-K3s Server和Agent配置](https://github.com/kingsd041/k3s-tutorial/blob/main/04-安装-K3s Server和Agent配置/README.md)

Public Reference：

[K3s Server 配置参考](https://docs.rancher.cn/docs/k3s/installation/install-options/server-config/_index)

[K3s Agent配置参考](https://docs.rancher.cn/docs/k3s/installation/install-options/agent-config/_index)

# 0 如何使用标志和环境变量

**理解 Server 节点的安装，以及注册 Agent 节点的步骤！**

Reference：https://docs.rancher.cn/docs/k3s/installation/install-options/how-to-flags/_index/

在整个 K3s 文档中，你会看到一些选项可以作为命令标志和环境变量传递进来。下面的例子展示了这些选项如何以两种方式传递。

## 示例 A： K3S_KUBECONFIG_MODE[#](https://docs.rancher.cn/docs/k3s/installation/install-options/how-to-flags/_index/#示例-a：-k3s_kubeconfig_mode)

允许写入 kubeconfig 文件的选项对于允许将 K3s 集群导入 Rancher 很有用。以下是传递该选项的两种方式。

使用标志 `--write-kubeconfig-mode 644`：

```shell
# 使用标志
$ curl -sfL https://get.k3s.io |sh -s - --write-kubeconfig-mode 644
```

使用环境变量 `K3S_KUBECONFIG_MODE`：

```shell
# 环境变量
$ curl -sfL https://get.k3s.io |K3S_KUBECONFIG_MODE="644" sh -s -
```

## 👑示例 B：INSTALL_K3S_EXEC[#](https://docs.rancher.cn/docs/k3s/installation/install-options/how-to-flags/_index/#示例-b：install_k3s_exec)

如果这个命令里没有指定为 server 或 agent，则如果设置了`K3S_URL`，则默认为 "`agent`"。如果没有设置，则默认为 "server"。

最后的 systemd 命令解析为这个环境变量和脚本参数的组合。为了说明这一点，以下命令的结果与注册一个没有 flannel 的 server 的行为相同：

```shell
# INSTALL_K3S_EXEC 有一个隐藏的功能：
# 可以判断该节点是 server 还是 agent 有所帮助
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--flannel-backend none" sh -s -
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --flannel-backend none" sh -s -
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server" sh -s - --flannel-backend none
curl -sfL https://get.k3s.io | sh -s - server --flannel-backend none
curl -sfL https://get.k3s.io | sh -s - --flannel-backend none
```

## 👑示例 C：CONFIG 文件[#](https://docs.rancher.cn/docs/k3s/installation/install-options/how-to-flags/_index/#示例-c-config-文件)

在安装 k3s 之前，你可以创建一个名为 config.yaml 的文件，其中包含与 CLI 标志相匹配的字段。该文件位于：`/etc/rancher/k3s/config.yaml`，K3s 在启动后会加载这个文件中的配置选项。

配置文件中的字段从匹配的 CLI 标志中删除开头的`--`。例如：

```shell
write-kubeconfig-mode: 644
token: "secret"
node-ip: 10.0.10.22,2a05:d012:c6f:4655:d73c:c825:a184:1b75
cluster-cidr: 10.42.0.0/16,2001:cafe:42:0::/56
service-cidr: 10.43.0.0/16,2001:cafe:42:1::/112
```

# 1 💡write-kubeconfig -- 将管理客户端的 kubeconfig 写入这个文件

```shell
# 默认的安装，不设置K3S_KUBECONFIG_OUTPUT环境变量
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN="rancher" sh -

$ ls /etc/rancher/k3s/k3s.yaml
/etc/rancher/k3s/k3s.yaml
$ cat /etc/rancher/k3s/k3s.yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: [......]
    server: https://127.0.0.1:6443
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: default
  user:
    client-certificate-data: [......]
    client-key-data: [......]

# 可以使用将k3s.yaml文件拷贝到~/.kube/config文件中
$ cp -av /etc/rancher/k3s/k3s.yaml ~/.kube/config
```

| **Flag**                             | **环境变量**          | **描述**                                                     |
| ------------------------------------ | --------------------- | ------------------------------------------------------------ |
| `--write-kubeconfig` value, -o value | K3S_KUBECONFIG_OUTPUT | 将管理客户端的 kubeconfig 写入这个文件。                     |
| `--write-kubeconfig`-mode value      | K3S_KUBECONFIG_MODE   | 使用这种[模式](https://en.wikipedia.org/wiki/Chmod) 写入 kubeconfig。允许写入 kubeconfig 文件的选项对于允许将 K3s 集群导入 Rancher 很有用。示例值为 644。 |

```shell
# write-kubeconfig
# 将管理客户端的kubeconfig写入这个文件
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  K3S_KUBECONFIG_OUTPUT=/root/.kube/config \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN="rancher" sh -

$ ls /root/.kube/config
/root/.kube/config

$ cat /root/.kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: [......]
    server: https://127.0.0.1:6443
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: default
  user:
    client-certificate-data: [......]
    client-key-data: [......]
```

# 2 💡使用 docker 作为容器运行时

| **Flag**                             | **默认值**                         | **描述**                                           |
| ------------------------------------ | ---------------------------------- | -------------------------------------------------- |
| `--docker`                           | N/A                                | 用 docker 代替 containerd                          |
| `--container-runtime-endpoint` value | N/A                                | 禁用嵌入式 containerd，使用替代的 CRI 实现。       |
| `--pause-image` value                | "docker.io/rancher/pause:3.1"      | 针对 containerd 或 Docker 的自定义 pause 镜像      |
| `--snapshotter` value                | N/A                                | 覆盖默认的 containerd 快照程序 (默认: "overlayfs") |
| `--private-registry` value           | "/etc/rancher/k3s/registries.yaml" | 私有镜像仓库配置文件                               |

在 K3s 节点上安装 Docker。可以使用 Rancher 的一个[Docker 安装脚本](https://github.com/rancher/install-docker)来安装 Docker：

```shell
curl https://releases.rancher.com/install-docker/19.03.sh | sh
```

国内用户，可以使用以下方法加速安装：

```shell
# 使用docker作为容器运行时
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC="--docker" \
  K3S_KUBECONFIG_OUTPUT=/root/.kube/config \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN="rancher" sh -

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
$ cat /etc/systemd/system/k3s.service | grep -A 3  "\<ExecStart\>"
ExecStart=/usr/local/bin/k3s \
    server \
        '--docker' \
```

扩展：

```shell
# 指定运行时工具
$ curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC="--container-runtime-endpoint containerd" \
  sh -

# 设置私有镜像仓库配置文件
# 默认配置文件: /etc/rancher/k3s/registries.yaml
$ curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC="--private-registry xxx" \
  sh -
```

# 3 💡针对多网卡主机安装 K3s 集群

## 3.1 K3s server:

```shell
# 针对多网卡主机安装K3s集群
# 默认k3s 集群部署是使用eth0 | ens33网卡的IP地址
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC="--advertise-address=192.168.10.30 --node-ip=192.168.10.30 --docker" \
  K3S_KUBECONFIG_OUTPUT=/root/.kube/config \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN="rancher" \
  sh -

# --advertise-address value	
# 默认值 node-external-ip/node-ip
# apiserver 用来向集群成员发布的 IP 地址

$ cat /var/lib/rancher/k3s/server/token
K10b9191902bd6d8c1d8923de73342aee780b9e01b060d100909c139d7f673dd3a0::server:rancher

# 使用的IP地址为192.168.10.30
$ kubectl get nodes -o wide
NAME   STATUS   ROLES                  AGE     VERSION         INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
k3s1   Ready    control-plane,master   5m      v1.21.14+k3s1   192.168.10.30   <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://19.3.15
```

## 3.2 K3s agent:

| **Flag**                    | **环境变量**    | **描述**                 |
| --------------------------- | --------------- | ------------------------ |
| `--node-ip` value, -i value | N/A             | 为节点发布的 IP 地址     |
| `--node-external-ip` value  | N/A             | 对外发布节点的 IP 地址   |
| `--resolv-conf` value       | K3S_RESOLV_CONF | Kubelet resolv.conf 文件 |
| `--flannel-iface` value     | N/A             | 覆盖默认的 flannel 接口  |
| `--flannel-conf `value      | N/A             | 覆盖默认的 flannel 文件  |

```shell
# K3S_URL : 指定K3s Master URL
# K3S_TOKEN : 指定K3s Master Token[cat /var/lib/rancher/k3s/server/token]
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  K3S_URL=https://192.168.10.30:6443 \
  K3S_TOKEN=rancher \
  K3S_KUBECONFIG_OUTPUT=/root/.kube/config \
  INSTALL_K3S_EXEC="--node-ip=192.168.10.31 --docker" \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  sh -

# 查看k3s agent日志
$ tail -n 200 -f /var/log/message

# 需要确保agent 和 server之间的连通性测试
# 在server节点中查看节点的情况
$ kubectl get nodes -o wide
NAME   STATUS   ROLES                  AGE     VERSION         INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
k3s1   Ready    control-plane,master   12m     v1.21.14+k3s1   192.168.10.30   <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://19.3.15
k3s2   Ready    <none>                 2m14s   v1.21.14+k3s1   192.168.10.31   <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://19.3.15
```

# 4 💡`--tls-san` -- 在 TLS 证书中添加其他主机名或 IP 作为主题备用名称

```shell
# 在公有云环境中使用较多，在K3s注册集群的时候会将一些IP地址自动收藏了
# 以及使用多个K3s Server搭建的K3s集群，并且拥有一个LB负载均衡器，就需要将LB的地址添加

# --tls-san
# 在TLS证书中添加其他主机名或IP作为主机备用名称
# 即在公网环境下允许通过公网IP访问控制、操作远程集群
# 或者部署多个Server并使用LB进行负责，就需要保留公网地
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC="--docker --tls-san 1.1.1.1" \
  K3S_KUBECONFIG_OUTPUT=/root/.kube/config \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN="rancher" \
  sh -

# 需要修改相应的 ~/.kube/config 文件的URL地址

# 获取配置
# k3s 会将信任的IP地址存放到k3s-serving的secrets中
$ kubectl get secrets -n kube-system k3s-serving
NAME          TYPE                DATA   AGE
k3s-serving   kubernetes.io/tls   2      117m

$ kubectl get secrets -n kube-system k3s-serving -o yaml
apiVersion: v1
data:
  tls.crt: [......]
  tls.key: [......]
kind: Secret
metadata:
  annotations:
    listener.cattle.io/cn-1.1.1.1: 1.1.1.1
    listener.cattle.io/cn-192.168.10.30: 192.168.10.30
    listener.cattle.io/cn-k8s-master: k8s-master
    listener.cattle.io/cn-kubernetes: kubernetes
    listener.cattle.io/cn-kubernetes.default: kubernetes.default
    listener.cattle.io/cn-kubernetes.default.svc: kubernetes.default.svc
    listener.cattle.io/cn-kubernetes.default.svc.cluster.local: kubernetes.default.svc.cluster.local
    [......]
type: kubernetes.io/tls

# 然后本机复制公网主节点对应的yaml文件即可本地操作了
$ scp root@1.1.1.1:/etc/rancher/k3s/k3s.yaml ~/.kube/config
```

# 5 👑修改`kube-apiserver`、`kube-scheduler` 、`kube-controller-manager`、 `kube-cloud-controller-manager`、 `kubelet`、 `kube-proxy` 参数

资料：[定制标志](https://docs.rancher.cn/docs/k3s/installation/install-options/server-config/_index#定制标志)

| **Flag**                                    | **描述**                                          |
| ------------------------------------------- | ------------------------------------------------- |
| `--etcd-arg` value                          | 自定义 etcd 进程的参数。                          |
| `--kube-apiserver-arg` value                | 自定义 kube-apiserver 进程的参数。                |
| `--kube-scheduler-arg` value                | 自定义 kube-scheduler 进程的参数。                |
| `--kube-controller-manager-arg` value       | 自定义 kube-controller-manager 进程的参数。       |
| `--kube-cloud-controller-manager-arg` value | 自定义 kube-cloud-controller-manager 进程的参数。 |

- kubelet-arg

中文文档：https://kubernetes.io/zh-cn/docs/reference/command-line-tools-reference/kubelet/

```shell
# 设置最多的Pod的数量
# --max-pods int32     默认值：110
# 修改启动的服务对应配置(调整节点的启动的最大Pod数量)
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC='--kubelet-arg=max-pods=200 --docker' \
  K3S_KUBECONFIG_OUTPUT=/root/.kube/config \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN="rancher" \
  sh -

$ kubectl describe nodes k8s-node01
Name:               k8s-node01
[......]
Annotations:        flannel.alpha.coreos.com/backend-data: {"VNI":1,"VtepMAC":"f2:b8:f3:fb:07:ca"}
                    flannel.alpha.coreos.com/backend-type: vxlan
                    flannel.alpha.coreos.com/kube-subnet-manager: true
                    flannel.alpha.coreos.com/public-ip: 10.0.0.31
                    k3s.io/hostname: k8s-node01
                    k3s.io/internal-ip: 10.0.0.31
                    k3s.io/node-args: ["server","--kubelet-arg","max-pods=200"]
                    k3s.io/node-config-hash: N5PDCMJFLSIUZI3X3OVB7E536YUHINIEJQBLGTKGJIWS573B63PA====
                    k3s.io/node-env: {"K3S_DATA_DIR":"/var/lib/rancher/k3s/data/630c40ff866a3db218a952ebd4fd2a5cfe1543a1a467e738cb46a2ad4012d6f1"}
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
[......]
Capacity:
  cpu:                4
  ephemeral-storage:  511750Mi
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3861264Ki
  pods:               200
Allocatable:
  cpu:                4
  ephemeral-storage:  509778329201
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3861264Ki
  pods:               200
[......]
```

- kube-apiserver-arg

中文文档：https://kubernetes.io/zh-cn/docs/reference/command-line-tools-reference/kube-apiserver/

```shell
# 保留给具有 NodePort 可见性的服务的端口范围。 不得与节点上的临时端口范围重叠。 例如："30000-32767"。范围的两端都包括在内。
# --service-node-port-range <形式为 'N1-N2' 的字符串>     默认值：30000-32767
# 修改启动的服务对应配置(调整服务启动的端口范围)
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC='--kube-apiserver-arg=service-node-port-range=40000-50000 --docker' \
  K3S_KUBECONFIG_OUTPUT=/root/.kube/config \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN="rancher" \
  sh -
```

- kube-proxy-arg

中文文档：https://kubernetes.io/zh-cn/docs/reference/command-line-tools-reference/kube-proxy/

```shell
# 使用哪种代理模式：'iptables'（仅 Linux）、'ipvs'（仅 Linux）、'kernelspace'（仅 Linux） 
# 或者 'userspace'（Linux/Windows, 已弃用）。 Linux 系统上的默认值是 'iptables'，
# Windows 系统上的默认值是 'userspace'。 如果配置文件由 --config 指定，则忽略此参数。
# --proxy-mode ProxyMode
# 修改启动的服务对应配置(使用ipvs作为服务调度工具)
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC='--kube-proxy-arg=proxy-mode=ipvs --docker' \
  K3S_KUBECONFIG_OUTPUT=/root/.kube/config \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN="rancher" \
  sh -

# 查看到网卡的信息
$ ip addr show 
[......]
32: kube-ipvs0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default
    link/ether 12:1c:14:d6:cd:f6 brd ff:ff:ff:ff:ff:ff
    inet 10.43.157.93/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.43.0.1/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.43.0.10/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever

##############################################################################
# kubelet-arg     --kubelet-arg
# kube-apiserver  --kube-apiserver-arg
# kube-proxy-arg  --kube-proxy-arg
# kube-proxy-arg  --kube-proxy-arg=proxy-mode=ipvs
```

# 6 👑`--data-dir` -- K3s 数据存储目录，默认为 `/var/lib/rancher/k3s` 或 `${HOME}/.rancher/k3s`(如果不是 root 用户)

```shell
# --data-dir
# 修改K3s数据存储目录
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC='--data-dir=/opt/k3s-data --docker' \
  K3S_KUBECONFIG_OUTPUT=/root/.kube/config \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN="rancher" \
  sh -

$ ls /opt/k3s-data/
agent  data  server
```

# 7 禁用组件 `--disable`

资料：[Kubernetes 组件](https://docs.rancher.cn/docs/k3s/installation/install-options/server-config/_index#kubernetes-组件)

| **Flag**                     | **描述**                                                     |
| ---------------------------- | ------------------------------------------------------------ |
| `--disable` value            | 不需要部署的组件，删除任何已部署的组件 (有效项目：`coredns, servicelb, traefik,local-storage, metrics-server`) |
| `--disable-scheduler`        | 禁用 Kubernetes 默认调度器                                   |
| `--disable-cloud-controller` | 禁用 k3s 默认云控制管理器                                    |
| `--disable-kube-proxy`       | 禁止运行 kube-proxy                                          |
| `--disable-network-policy`   | 禁用 K3S 默认网络策略控制器                                  |

```shell
# 默认安装
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -
$ ls /var/lib/rancher/k3s/server/manifests
ccm.yaml  coredns.yaml  local-storage.yaml  metrics-server  rolebindings.yaml  traefik.yaml

############################################################################################
# 禁用组件
# --disable traefik
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC='--disable traefik' \
  sh -

# 自己加自己需要的服务
$ ls /var/lib/rancher/k3s/server/manifests
ccm.yaml  coredns.yaml  local-storage.yaml  metrics-server  rolebindings.yaml

$ kubectl get pods -A | grep traefik
```

# 8 添加 label 和 taint

```shell
# 设置完污点后，无法通过k3s命令进行修改
# 只能通过kubectl进行修改污点
# 添加label和taint标识
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC='--node-label foo=bar,hello=world --node-taint key1=value1:NoExecute --docker' \
  K3S_KUBECONFIG_OUTPUT=/root/.kube/config \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN="rancher" \
  sh -

# 查看一下
$ kubectl describe nodes 
```

# 9 存储类选项

| **Flag**                             | **描述**                 |
| ------------------------------------ | ------------------------ |
| `--default-local-storage-path value` | 本地存储类的默认存储路径 |

```shell
$ kubectl get sc # kubectl get storageclass
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  10m

# 查看 local-storage.yaml
$ cat /var/lib/rancher/k3s/server/manifests/local-storage.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: local-path-provisioner-service-account
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: local-path-provisioner-role
rules:
- apiGroups: [""]
  resources: ["nodes", "persistentvolumeclaims", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["endpoints", "persistentvolumes", "pods"]
  verbs: ["*"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "patch"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: local-path-provisioner-bind
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: local-path-provisioner-role
subjects:
- kind: ServiceAccount
  name: local-path-provisioner-service-account
  namespace: kube-system
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: local-path-provisioner
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: local-path-provisioner
  template:
    metadata:
      labels:
        app: local-path-provisioner
    spec:
      priorityClassName: "system-node-critical"
      serviceAccountName: local-path-provisioner-service-account
      tolerations:
          - key: "CriticalAddonsOnly"
            operator: "Exists"
          - key: "node-role.kubernetes.io/control-plane"
            operator: "Exists"
            effect: "NoSchedule"
          - key: "node-role.kubernetes.io/master"
            operator: "Exists"
            effect: "NoSchedule"
      containers:
      - name: local-path-provisioner
        image: rancher/local-path-provisioner:v0.0.21
        imagePullPolicy: IfNotPresent
        command:
        - local-path-provisioner
        - start
        - --config
        - /etc/config/config.json
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config/
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
      volumes:
        - name: config-volume
          configMap:
            name: local-path-config
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: local-path-config
  namespace: kube-system
data:
  config.json: |-
    {
      "nodePathMap":[
      {
        "node":"DEFAULT_PATH_FOR_NON_LISTED_NODES",
        "paths":["/opt/k3s-data/storage"]
      }
      ]
    }
  setup: |-
    #!/bin/sh
    while getopts "m:s:p:" opt
    do
        case $opt in
            p)
            absolutePath=$OPTARG
            ;;
            s)
            sizeInBytes=$OPTARG
            ;;
            m)
            volMode=$OPTARG
            ;;
        esac
    done
    mkdir -m 0777 -p ${absolutePath}
    chmod 701 ${absolutePath}/..
  teardown: |-
    #!/bin/sh
    while getopts "m:s:p:" opt
    do
        case $opt in
            p)
            absolutePath=$OPTARG
            ;;
            s)
            sizeInBytes=$OPTARG
            ;;
            m)
            volMode=$OPTARG
            ;;
        esac
    done
    rm -rf ${absolutePath}
  helperPod.yaml: |-
    apiVersion: v1
    kind: Pod
    metadata:
      name: helper-pod
    spec:
      containers:
      - name: helper-pod
        image: rancher/mirrored-library-busybox:1.34.1
```

# 10 数据库选项

资料：[数据库](https://docs.rancher.cn/docs/k3s/installation/install-options/server-config/_index#数据库)

| **Flag**                              | **环境变量**           | **描述**                                                     |
| ------------------------------------- | ---------------------- | ------------------------------------------------------------ |
| `--datastore-endpoint` value          | K3S_DATASTORE_ENDPOINT | 指定 etcd、Mysql、Postgres 或 Sqlite（默认）数据源名称。     |
| `--datastore-cafile` value            | K3S_DATASTORE_CAFILE   | TLS 证书授权文件，用于确保数据存储后端通信的安全。           |
| `--datastore-certfile` value          | K3S_DATASTORE_CERTFILE | TLS 认证文件，用于确保数据存储后端通信的安全。               |
| `--datastore-keyfile` value           | K3S_DATASTORE_KEYFILE  | 用于保护数据存储后端通信的 TLS 密钥文件。                    |
| `--etcd-expose-metrics`               | N/A                    | 将 etcd 指标公开给客户端界面。(默认为 false)                 |
| `--etcd-disable-snapshots`            | N/A                    | 禁用自动 etcd 快照                                           |
| `--etcd-snapshot-name` value          | N/A                    | 设置 etcd 快照的基本名称。默认值: etcd-snapshot              |
| `--etcd-snapshot-schedule-cron` value | N/A                    | cron 规范中的快照间隔时间。 例如。每 5 小时 "* */5 * * *" (默认值: "0 */12 * * *") |
| `--etcd-snapshot-retention` value     | N/A                    | 要保留的快照数量 (默认值: 5)                                 |
| `--etcd-snapshot-dir` value           | N/A                    | 保存数据库快照的目录. (默认 location: ${data-dir}/db/snapshots) |
| `--etcd-s3`                           | N/A                    | 启用备份到 S3                                                |
| `--etcd-s3-endpoint` value            | N/A                    | S3 endpoint url (默认值: "s3.amazonaws.com")                 |
| `--etcd-s3-endpoint-ca` value         | N/A                    | S3 自定义 CA 证书连接到 S3 endpoint                          |
| `--etcd-s3-skip-ssl-verify`           | N/A                    | 禁用 S3 的 SSL 证书验证                                      |
| `--etcd-s3-access-key` value          | AWS_ACCESS_KEY_ID      | S3 access key                                                |
| `--etcd-s3-secret-key` value          | AWS_SECRET_ACCESS_KEY  | S3 secret key                                                |
| `--etcd-s3-bucket` value              | N/A                    | S3 bucket 名称                                               |
| `--etcd-s3-region` value              | N/A                    | S3 region / bucket 位置 (可选) 默认值: "us-east-1")          |
| `--etcd-s3-folder` value              | N/A                    | S3 文件夹                                                    |

```shell
# 指定数据源名称
# 标志位: --datastore-endpoint&nbsp;value
# 环境变量: K3S_DATASTORE_ENDPOINT
$ curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC='--datastore-endpoint etcd' \
  K3S_KUBECONFIG_OUTPUT=/root/.kube/config \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN="rancher" \
  sh -
# cron规范中的快照间隔时间
# --etcd-snapshot-schedule-cron&nbsp;value
$ curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC='--etcd-snapshot-schedule-cron 0 \*/12 \* \* \*' \
  K3S_KUBECONFIG_OUTPUT=/root/.kube/config \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN="rancher" \
  sh -
```

# 11 网络选项

| **Flag**                          | **默认值**      | **描述**                                                    |
| --------------------------------- | --------------- | ----------------------------------------------------------- |
| `--cluster-cidr` value            | "10.42.0.0/16"  | 用于 Pod IP 的网络 CIDR                                     |
| `--service-cidr` value            | "10.43.0.0/16"  | 用于 service IP 的网络 CIDR                                 |
| `--service-node-port-range` value | "30000-32767"   | 为具有 NodePort 可见性的服务保留的端口范围                  |
| `--cluster-dns` value             | "10.43.0.10"    | 用于 coredns 服务的集群 IP。应该在您的service-cidr范围内    |
| `--cluster-domain` value          | "cluster.local" | 集群域名                                                    |
| `--flannel-backend` value         | "vxlan"         | 'none', 'vxlan', 'ipsec', 'host-gw', 或 'wireguard'中的一个 |

# 12 日志选项

| **Flag**                | **默认值** | **描述**                                                |
| ----------------------- | ---------- | ------------------------------------------------------- |
| `--debug`               | N/A        | 开启调试日志                                            |
| `-v` value              | 0          | 日志级别详细程度的数字                                  |
| `--vmodule` value       | N/A        | 以逗号分隔的 pattern=N 设置列表，用于文件过滤的日志记录 |
| `--log value`, -l value | N/A        | 记录到文件                                              |
| `--alsologtostderr`     | N/A        | 记录到标准错误输出和文件（如果设置）。                  |

```shell
# 默认情况下k3s输出的日志路径
ubuntu: /var/log/syslogd
centos: /var/log/messages
```

# 13 K3s Server CLI 帮助

如果一个选项出现在下面的括号里，例如`[$K3S_TOKEN]`，则意味着该选项可以作为该名称的环境变量传递进来。

```shell
NAME:
   k3s server - 运行K3s server

USAGE:
   k3s server [选项]

选项:
   --config FILE, -c FILE                     (config) 从FILE加载配置（默认："/etc/rancher/k3s/config.yaml" ） [$K3S_CONFIG_FILE]
   --debug                                    (logging) 开启debug调试日志 [$K3S_DEBUG]
   -v value                                   (logging) 日志级别详细程度的数字 (默认: 0)
   --vmodule value                            (logging) 以逗号分隔的pattern=N设置列表，用于文件过滤的日志记录
   --log value, -l value                      (logging) 记录到文件
   --alsologtostderr                          (logging) 记录到标准错误输出和文件（如果设置)
   --bind-address value                       (listener) k3s绑定地址 (默认: 0.0.0.0)
   --https-listen-port value                  (listener) HTTPS监听端口 (默认: 6443)
   --advertise-address value                  (listener) apiserver用来向集群成员发布的IP地址 (默认: node-external-ip/node-ip)
   --advertise-port value                     (listener) apiserver用于通告集群成员的端口 (默认: listen-port) (默认: 0)
   --tls-san value                            (listener) 在TLS证书中添加其他主机名或IP作为主题备用名称
   --data-dir value, -d value                 (data) 存放数据的目录 默认 /var/lib/rancher/k3s or ${HOME}/.rancher/k3s(如果不是root用户)
   --cluster-cidr value                       (networking) 用于Pod IP的网络CIDR (默认: "10.42.0.0/16")
   --service-cidr value                       (networking) 用于service IP的网络CIDR (默认: "10.43.0.0/16")
   --service-node-port-range value            (networking) 为具有 NodePort 可见性的服务保留的端口范围 (默认: "30000-32767")
   --cluster-dns value                        (networking) 用于coredns服务的集群IP。应该在您的`service-cidr`范围内 (默认: 10.43.0.10)
   --cluster-domain value                     (networking) 集群域名 (默认: "cluster.local")
   --flannel-backend value                    (networking) 'none', 'vxlan', 'ipsec', 'host-gw', 或 'wireguard'中的一个 (默认: "vxlan")
   --token value, -t value                    (cluster) 用于将server或agent加入集群的共享secret。 [$K3S_TOKEN]
   --token-file value                         (cluster) 包含cluster-secret/token的文件 [$K3S_TOKEN_FILE]
   --write-kubeconfig value, -o value         (client) 将管理客户端的kubeconfig写入这个文件。 [$K3S_KUBECONFIG_OUTPUT]
   --write-kubeconfig-mode value              (client) 用这种模式编写kubeconfig，例如：644 [$K3S_KUBECONFIG_MODE]
   --etcd-arg value                           (flags) 自定义etcd进程的参数
   --kube-apiserver-arg value                 (flags) 自定义kube-apiserver进程的参数
   --kube-scheduler-arg value                 (flags) 自定义kube-scheduler进程的参数
   --kube-controller-manager-arg value        (flags) 自定义kube-controller-manager进程的参数
   --kube-cloud-controller-manager-arg value  (flags) 自定义kube-cloud-controller-manager进程的参数
   --datastore-endpoint value                 (db) 指定 etcd、Mysql、Postgres 或 Sqlite（默认）数据源名称 [$K3S_DATASTORE_ENDPOINT]
   --datastore-cafile value                   (db) TLS证书授权文件，用于确保数据存储后端通信的安全 [$K3S_DATASTORE_CAFILE]
   --datastore-certfile value                 (db) TLS认证文件，用于确保数据存储后端通信的安全 [$K3S_DATASTORE_CERTFILE]
   --datastore-keyfile value                  (db) 用于保护数据存储后端通信的TLS密钥文件 [$K3S_DATASTORE_KEYFILE]
   --etcd-expose-metrics                      (db) 将 etcd 指标公开给客户端界面。(默认为 false)
   --etcd-disable-snapshots                   (db) 禁用自动 etcd 快照
   --etcd-snapshot-name value                 (db) 设置 etcd 快照的基本名称。默认值: etcd-snapshot
   --etcd-snapshot-schedule-cron value        (db) cron 规范中的快照间隔时间。 例如。每 5 小时 "\* \*/5 \* \* \*" (默认值: "0 \*/12 \* \* \*")
   --etcd-snapshot-retention value            (db) 要保留的快照数量 (默认值: 5)
   --etcd-snapshot-dir value                  (db) 保存数据库快照的目录. (默认 location: ${data-dir}/db/snapshots)
   --etcd-s3                                  (db) 启用备份到 S3
   --etcd-s3-endpoint value                   (db) S3 endpoint url (默认值: "s3.amazonaws.com")
   --etcd-s3-endpoint-ca value                (db) S3 自定义 CA 证书连接到 S3 endpoint
   --etcd-s3-skip-ssl-verify                  (db) 禁用 S3 的 SSL 证书验证
   --etcd-s3-access-key value                 (db) S3 access key [$AWS_ACCESS_KEY_ID]
   --etcd-s3-secret-key value                 (db) S3 secret key [$AWS_SECRET_ACCESS_KEY]
   --etcd-s3-bucket value                     (db) S3 bucket 名称
   --etcd-s3-region value                     (db) S3 region / bucket 位置 (可选) 默认值: "us-east-1")
   --etcd-s3-folder value                     (db) S3 文件夹
   --default-local-storage-path value         (storage) 本地存储类的默认存储路径
   --disable value                            (components) 不需要部署的组件，删除任何已部署的组件 (有效项目：coredns, servicelb, traefik,local-storage, metrics-server)
   --disable-scheduler                        (components) 禁用Kubernetes默认调度器
   --disable-cloud-controller                 (components) 禁用k3s默认云控制管理器
   --disable-kube-proxy                       (components) 禁止运行 kube-proxy
   --disable-network-policy                   (components) 用K3S默认网络策略控制器
   --node-name value                          (agent/node) 节点名称 [$K3S_NODE_NAME]
   --with-node-id                             (agent/node) 将ID附加到节点名称
   --node-label value                         (agent/node) 用一组标签注册和启动kubelet
   --node-taint value                         (agent/node) 用一组污点注册kubelet
   --image-credential-provider-bin-dir value  (agent/node) 凭证提供程序插件二进制文件所在目录的路径（默认："/var/lib/rancher/credentialprovider/bin"）
   --image-credential-provider-config value   (agent/node) 凭证提供者插件配置文件的路径 (默认： "/var/lib/rancher/credentialprovider/config.yaml")
   --docker                                   (agent/runtime) 用docker代替containerd
   --container-runtime-endpoint value         (agent/runtime) 禁用嵌入式containerd，使用替代的CRI实现
   --pause-image value                        (agent/runtime) 针对containerd或Docker的自定义pause镜像 (默认: "docker.io/rancher/pause:3.1")
   --snapshotter value                        (agent/runtime) 覆盖默认的 containerd 快照程序 (默认: "overlayfs")
   --private-registry value                   (agent/runtime) 私有镜像仓库配置文件 (默认: "/etc/rancher/k3s/registries.yaml")
   --node-ip value, -i value                  (agent/networking) 为节点发布的IP地址
   --node-external-ip value                   (agent/networking) 对外发布节点的IP地址
   --resolv-conf value                        (agent/networking) Kubelet resolv.conf 文件 [$K3S_RESOLV_CONF]
   --flannel-iface value                      (agent/networking) 覆盖默认的flannel接口
   --flannel-conf value                       (agent/networking) 覆盖默认的flannel文件
   --kubelet-arg value                        (agent/flags) 自定义kubelet进程的参数
   --kube-proxy-arg value                     (agent/flags) 自定义kube-proxy进程的参数
   --protect-kernel-defaults                  (agent/node) 内核调优行为。如果设置，如果内核可调参数与 kubelet 默认值不同，则会出错。
   --rootless                                 (experimental) 运行 rootless
   --agent-token value                        (experimental/cluster) 用于将agent加入集群但不用于server的共享密钥 [$K3S_AGENT_TOKEN]
   --agent-token-file value                   (experimental/cluster) 包含agent secret的文件 [$K3S_AGENT_TOKEN_FILE]
   --server value, -s value                   (experimental/cluster) 要连接的k3s server，用于加入集群 [$K3S_URL]
   --cluster-init                             (experimental/cluster) 初始化为新的集群master [$K3S_CLUSTER_INIT]
   --cluster-reset                            (experimental/cluster) 忽略所有节点，成为一个新集群的集群master [$K3S_CLUSTER_RESET]
   --cluster-reset-restore-path value         (db) 要恢复的快照文件的路径
   --secrets-encryption                       (experimental) 启用Secret加密
   --system-default-registry value            (image) 用于所有系统镜像的私有注册表 [$K3S_SYSTEM_DEFAULT_REGISTRY]
   --selinux                                  (agent/node) 在 containerd 中启用 SELinux [$K3S_SELINUX]
   --lb-server-port value                     (agent/node) 客户端负载均衡器的本地端口。如果 supervisor 和 apiserver 不在同一个主机，则 apiserver 客户端负载均衡器也将使用比该端口小 1 的额外端口。(默认：6444) [$K3S_LB_SERVER_PORT]
   --no-flannel                               (deprecated) 使用 --flannel-backend=none
   --no-deploy value                          (deprecated) 不需要部署的组件 (有效选项: coredns, servicelb, traefik, local-storage, metrics-server)
   --cluster-secret value                     (deprecated) 使用 --token [$K3S_CLUSTER_SECRET]
```