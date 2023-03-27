Video Reference：[12-升级k3s集群](https://www.bilibili.com/video/BV16R4y1g77f/?spm_id_from=333.788&vd_source=9560c118fae1db9638f05a6ba2527085)

GitHub README.md：[12-升级k3s集群](https://github.com/kingsd041/k3s-tutorial/tree/main/12-升级k3s集群)

**手动升级 + 自动升级**

# 1 升级 K3s 集群

当升级 `K3s` 时，`K3s` 服务会重启或停止，但 `K3s` 容器会继续运行。 要停止所有的 `K3s` 容器并重置容器的状态，可以使用 `k3s-killall.sh` 脚本。 killall 脚本清理容器、`K3s` 目录和网络组件，同时也删除了 `iptables` 链和所有相关规则。集群数据不会被删除。当重启 `K3s` 集群时会进行重建 `iptables` 链和所有相关的规则。

范例：`k3s-killall.sh` 的脚本内容( 实际环境很少使用 )

```shell
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
```

## 1.1 手动升级

你可以通过使用安装脚本升级 K3s，或者手动安装所需版本的二进制文件。

注意： 升级时，先逐个升级 server 节点，然后再升级其他 agent 节点。可以保证服务的高可用性。

### Channels 说明

通过安装脚本或使用我们的[自动升级](http://docs.rancher.cn/docs/k3s/upgrades/basic/_index)功能进行的升级可以绑定到不同的发布 channels。以下是可用的 channels。

| **CHANNEL**  | **描 述**                                                    |
| ------------ | ------------------------------------------------------------ |
| stable       | (默认)稳定版建议用于生产环境。这些版本已经过一段时间的社区强化。 |
| latest       | 推荐使用最新版本尝试最新的功能。 这些版本还没有经过社区强化。 |
| v1.19 (例子) | 每一个支持的 Kubernetes 次要版本都有一个发布 channel，它们分别是`v1.19`、`v1.20`和`v1.21`。这些 channel 会选择最新的可用补丁版本，不一定是稳定版本。 |

对于详细的最新 channels 列表，您可以访问[k3s channel 服务 API](https://update.k3s.io/v1-release/channels)。关于 channels 工作的更多技术细节，请参见[channelserver 项目](https://github.com/rancher/channelserver)。建议使用 stable 稳定版的版本。

### 演示环境

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

################################################################################################################################################################
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

# 查看节点的状态
$ kubectl get nodes
NAME   STATUS   ROLES                  AGE    VERSION
k3s1   Ready    control-plane,master   6m1s   v1.21.14+k3s1
k3s2   Ready    <none>                 108s   v1.21.14+k3s1
```

### 使用安装脚本升级 K3s

要从旧版本升级 K3s，你可以使用`**相同的标志(参数)**`重新运行安装脚本：

- [ 例如：当使用Docker作为容器运行时环境，就需要在做升级和降低K3s集群的版本时，也需要添加Docker运行时环境的参数，否则会使用Containerd作为容器运行时环境CRI，可以通过`/etc/systemd/system/k3s.service`的`ExecStart`查看相应的参数 ]
- 升级到最新 stable 版本

```bash
$ curl -sfL https://get.k3s.io | \
  INSTALL_K3S_MIRROR=cn \
  sh -
```

- 升级到 latest 版本

```bash
# INSTALL_K3S_CHANNEL 默认的参数是 stable
curl -sfL https://get.k3s.io | \
INSTALL_K3S_MIRROR=cn \
INSTALL_K3S_CHANNEL=latest sh - 

# K3s Master执行
$ curl -sfL https://get.k3s.io | \
  INSTALL_K3S_MIRROR=cn INSTALL_K3S_EXEC="--docker" \
  INSTALL_K3S_CHANNEL=latest sh - 

$ kubectl get nodes
NAME       STATUS     ROLES                  AGE   VERSION
k3s2       Ready      <none>                 10m   v1.21.14+k3s1
k3s1       Ready      control-plane,master   91s   v1.26.2+k3s1

# K3s Worker执行
$ curl -sfL https://get.k3s.io | \
  K3S_URL=https://10.0.0.51:6443 \
  K3S_TOKEN=rancher INSTALL_K3S_MIRROR=cn INSTALL_K3S_EXEC="--docker" \
  INSTALL_K3S_CHANNEL=latest sh - 

$ kubectl get nodes
NAME       STATUS     ROLES                  AGE   VERSION
k3s2       Ready      <none>                 92s   v1.26.2+k3s1
k3s1       Ready      control-plane,master   91s   v1.26.2+k3s1
```

- 升级到 v1.20 的最新版本

```shell
$ curl -sfL https://get.k3s.io | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_CHANNEL="v1.20" sh - 
```

- 升级到指定版本

```shell
$ curl -sfL https://get.k3s.io | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_VERSION=vX.Y.Z-rc1 sh -
```

### 使用二进制文件手动升级 K3s

1. 从[发布](https://github.com/rancher/k3s/releases)下载所需版本的 K3s 二进制文件
2. 将下载的二进制文件复制到`/usr/local/bin/k3s`（或您所需的位置）
3. 停止旧的 K3s 二进制文件
4. 启动新的 K3s 二进制文件

```shell
# 下载Github中的k3s二进制
$ wget https://github.com/k3s-io/k3s/releases/download/v1.26.3-rc2%2Bk3s1/k3s
$ mv ./k3s /usr/local/bin/k3s && chmod +x /usr/local/bin/k3s

# 重启K3s集群即可
$ systemctl restart k3s.service
# 查看Node节点的K3s版本
$ kubectl get nodes
NAME       STATUS   ROLES                  AGE   VERSION
k3s2       Ready    <none>                 81m   v1.26.2+k3s1
k3s1       Ready    control-plane,master   86m   v1.26.3-rc2+k3s1
```

## 1.2 自动升级

💡注意： 此功能从 v1.17.4+k3s1 开始提供支持。

你可以使用 Rancher 的 `system-upgrad-controller` 来管理 K3s 集群升级。这是一种 Kubernetes 原生的集群升级方法。它利用[自定义资源定义(CRD)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#custom-resources)、计划和[控制器](https://kubernetes.io/docs/concepts/architecture/controller/)，根据配置的计划安排升级。

控制器通过监控计划和选择要在其上运行升级[job](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/) 的节点来调度升级。计划通过[标签选择器](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)定义哪些节点应该升级。当一个 job 成功运行完成后，控制器会给它运行的节点打上相应的标签。

关于 `system-upgrade-controller` 的设计和架构或其与 K3s 集成的更多细节，请参见以下 Git 仓库：

- [system-upgrade-controller](https://github.com/rancher/system-upgrade-controller)
- [k3s-upgrade](https://github.com/rancher/k3s-upgrade)

要以这种方式进行自动升级，你必须：

1. 将 `system-upgrade-controller` 安装到您的集群中
2. 配置计划

### 1.2.1 演示环境

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

################################################################################################################################################################
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

# 查看节点的状态
$ kubectl get nodes
NAME   STATUS   ROLES                  AGE     VERSION
k3s1   Ready    control-plane,master   2m58s   v1.21.14+k3s1
k3s2   Ready    <none>                 2m12s   v1.21.14+k3s1
```

### 1.2.2 安装 system-upgrade-controller

System-upgrade-controller 可以作为 deployment 安装到您的集群中。Deployment 需要一个 service-account、clusterRoleBinding 和一个 configmap。要安装这些组件，请运行以下命令:

```shell
# 旧版本的Yaml文件
kubectl apply -f https://github.com/rancher/system-upgrade-controller/releases/download/v0.6.2/system-upgrade-controller.yaml

# 新版本的Yaml文件
kubectl apply -f https://github.com/rancher/system-upgrade-controller/releases/download/v0.10.0/system-upgrade-controller.yaml
```

查看`system-upgrade-controller`的状态

```shell
$ kubectl get all --namespace system-upgrade
NAME                                             READY   STATUS    RESTARTS   AGE
pod/system-upgrade-controller-77d5494d7f-hhcg6   1/1     Running   0          49s

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/system-upgrade-controller   1/1     1            1           49s

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/system-upgrade-controller-77d5494d7f   1         1         1       49s
```

### 1.2.3 配置计划

建议您最少创建两个计划：升级 server（master）节点的计划和升级 agent（worker）节点的计划。根据需要，您可以创建其他计划来控制跨节点的滚动升级。以下两个示例计划将把您的集群升级到 `K3s v1.26.2+k3s1`。创建计划后，控制器将接收这些计划并开始升级您的集群。

```yaml
# k3s-upgrade-node.yaml
# Server plan
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: server-plan
  # 使用命名空间，要与system-upgrade-controller放在同一个命名空间中
  namespace: system-upgrade 
spec:
  # 同时升级的最大节点数
  concurrency: 1
  # 将升级的节点设置为不可用的状态
  cordon: true
  # 节点选择器
  nodeSelector:
    # kubectl describe nodes [NODE_NAME] 查看节点的标签情况
    matchExpressions:
    - key: node-role.kubernetes.io/master
      operator: In
      values:
      - "true"
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/k3s-upgrade
  # 升级的版本
  version: v1.26.2+k3s1

---
# Agent plan
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: agent-plan
  namespace: system-upgrade
spec:
  concurrency: 1
  cordon: true
  # 节点选择器
  nodeSelector:
    matchExpressions:
    - key: node-role.kubernetes.io/master
      operator: DoesNotExist
  # 需要等待k3s的server-plan计划执行完毕后才能执行agent-plan计划
  prepare:
    args:
    - prepare
    - server-plan
    image: rancher/k3s-upgrade
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/k3s-upgrade
  # 升级的版本
  version: v1.26.2+k3s1
```

关于这些计划，有几个重要的事情需要提醒：

- 首先，必须在部署控制器的同一命名空间中创建计划。
- 其次，`concurrency`字段表示可以同时升级多少个节点。
- 第三，`server-plan`通过指定一个标签选择器来选择带有`node-role.kubernetes.io/master`标签的节点，从而锁定 server 节点。agent-plan通过指定一个标签选择器来选择没有该标签的节点，以 agent 节点为目标。
- 第四，`agent-plan`中的 prepare 步骤会使该计划等待`server-plan`完成后再执行升级 jobs。

```shell
# 引用相关的Yaml资源文件
$ kubectl create -f k3s-upgrade-node.yaml
plan.upgrade.cattle.io/server-plan created
plan.upgrade.cattle.io/agent-plan created

# 查看Plan的资源情况
$ kubectl get plan --namespace system-upgrade
NAME          IMAGE                 CHANNEL   VERSION
agent-plan    rancher/k3s-upgrade             v1.26.2+k3s1
server-plan   rancher/k3s-upgrade             v1.26.2+k3s1

$ kubectl get job --namespace system-upgrade
NAME                                                              COMPLETIONS   DURATION   AGE
apply-server-plan-on-k3s1-with-fffd7522514e0137ae9fc76f35-0024d   1/1           46s        48s
apply-agent-plan-on-k3s2-with-fffd7522514e0137ae9fc76f35a-eebf5   0/1           48s        48s

# 查看Pod的情况(apply-server-plan和apply-agent-plan全部执行成功)
$ kubectl get pod --namespace system-upgrade
NAME                                                              READY   STATUS      RESTARTS   AGE
system-upgrade-controller-77d5494d7f-hhcg6                        1/1     Running     0          10m
apply-server-plan-on-k3s1-with-fffd7522514e0137ae9fc76f35-rzgmq   0/1     Completed   0          98s
apply-agent-plan-on-k3s2-with-fffd7522514e0137ae9fc76f35a-dq29m   0/1     Completed   0          98s

# 查看Node节点的版本
$ kubectl get nodes
NAME   STATUS   ROLES                  AGE   VERSION
k3s1   Ready    control-plane,master   13m   v1.26.2+k3s1
k3s2   Ready    <none>                 12m   v1.26.2+k3s1
```

- 第五，两个计划的version字段都设置为 `v1.26.2+k3s1`。或者，你可以省略 `version` 字段，将 `channel` 字段设置为解析到 K3s 版本的 URL。这将导致控制器监控该 URL，并在它解析到新版本时随时升级集群。这与 [release channels](https://github.com/kingsd041/k3s-tutorial/blob/main/docs/k3s/upgrades/basic/_index#发布-channels) 配合得很好。因此，你可以用下面的 channel 配置你的计划，以确保你的集群总是自动升级到 K3s 的最新稳定版本。

查看相关的文件内容

```yaml
apiVersion: upgrade.cattle.io/v1
kind: Plan
...
spec:
  ...
  channel: https://update.k3s.io/v1-release/channels/stable
```

演示环境

```shell
$ kubectl delete -f k3s-upgrade-node.yaml
plan.upgrade.cattle.io "server-plan" deleted
plan.upgrade.cattle.io "agent-plan" deleted

# 将 version 字段删除后替换成 channel 字段格式
$ vim k3s-upgrade-node.yaml

$ kubectl create -f k3s-upgrade-node.yaml
plan.upgrade.cattle.io/server-plan created
plan.upgrade.cattle.io/agent-plan created

# 查看Node节点的k3s版本
$ kubectl get nodes
NAME   STATUS                     ROLES                  AGE   VERSION
k3s1   Ready,SchedulingDisabled   control-plane,master   24m   v1.25.7+k3s1
k3s2   Ready                      <none>                 23m   v1.26.2+k3s1

# -->
$ kubectl get nodes
NAME   STATUS                     ROLES                  AGE   VERSION
k3s1   Ready                      control-plane,master   24m   v1.25.7+k3s1
k3s2   Ready,SchedulingDisabled   <none>                 24m   v1.26.2+k3s1

# 最终的Node节点的情况
$ kubectl get nodes
NAME   STATUS   ROLES                  AGE   VERSION
k3s1   Ready    control-plane,master   25m   v1.25.7+k3s1
k3s2   Ready    <none>                 24m   v1.25.7+k3s1
```

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679940592104-daa2f05e-341d-4973-9e40-8be1acd8435a.png)

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679940610721-d7cb46eb-93c3-4faf-b6fe-87fa17ba59d3.png)

如上所述，一旦控制器检测到计划已创建，升级就会立即开始。更新计划将使控制器重新评估计划并确定是否需要再次升级。

🎓当进行对 K3s 集群服务进行停止或者宕掉，用 K3s 创建的 Pod 以及其他的资源对象是不会有任何的影响，是跟业务的 Pod 不会有任何的影响。

您可以通过 `kubectl` 查看 `plans` 和 `jobs` 来监控升级的进度：

```shell
# 查看 plans 来监控升级的进度
$ kubectl -n system-upgrade get plans -o yaml
# 查看 jobs 来监控升级的进度
$ kubectl -n system-upgrade get jobs -o yaml

# 可以通过Pod的状态进行排查
$ kubectl get pod -n system-upgrade
```

## 1.3 k3s-upgrade.sh 自动升级的脚本

GitHub 地址：https://github.com/k3s-io/k3s-upgrade/blob/master/scripts/upgrade.sh

```shell
#!/bin/sh -xe

info()
{
    echo '[INFO] ' "$@"
}

fatal()
{
    echo '[ERROR] ' "$@" >&2
    exit 1
}

get_k3s_process_info() {
  K3S_PID=$(ps -ef | grep -E "k3s .*(server|agent)" | grep -E -v "(init|grep|channelserver|supervise-daemon)" | awk '{print $2}')

  if [ -z "$K3S_PID" ]; then
    fatal "K3s is not running on this server"
  fi

  if [ "$(echo $K3S_PID | wc -w)" != "1" ]; then
    for PID in $K3S_PID; do
      ps -fp $PID || true
    done
    fatal "Found multiple K3s pids"
  fi

  K3S_PPID=$(ps -p $K3S_PID -o ppid= | awk '{print $1}')
  info "K3S binary is running with pid $K3S_PID, parent pid $K3S_PPID"

  # When running with the --log flag, the 'k3s server|agent' process is nested under a 'k3s init' process.
  # If the parent pid is not 1 (init/systemd) then we are nested and need to operate against that 'k3s init' pid instead.
  # Make sure that the parent pid is actually k3s though, as openrc systems may run k3s under supervise-daemon instead of
  # as a child process of init.
  if [ "$K3S_PPID" != "1" ] && grep -qs k3s /host/proc/${K3S_PPID}/cmdline; then
    K3S_PID="${K3S_PPID}"
  fi

  # When running in k3d, k3s will be pid 1 and is always at /bin/k3s
  if [ "$K3S_PID" == "1" ]; then
    K3S_BIN_PATH="/bin/k3s"
  else
    K3S_BIN_PATH=$(awk 'NR==1 {print $1}' /host/proc/${K3S_PID}/cmdline)
  fi

  if [ -z "$K3S_BIN_PATH" ]; then
    fatal "Failed to fetch the k3s binary path from pid $K3S_PID"
  fi
  return
}

replace_binary() {
  NEW_BINARY="/opt/k3s"
  FULL_BIN_PATH="/host$K3S_BIN_PATH"

  if [ ! -f "$NEW_BINARY" ]; then
    fatal "The new binary $NEW_BINARY doesn't exist"
  fi

  info "Comparing old and new binaries"
  BIN_CHECKSUMS="$(sha256sum $NEW_BINARY $FULL_BIN_PATH)"

  if [ "$?" != "0" ]; then
    fatal "Failed to calculate binary checksums"
  fi

  BIN_COUNT="$(echo "${BIN_CHECKSUMS}" | awk '{print $1}' | uniq | wc -l)"
  if [ "$BIN_COUNT" == "1" ]; then
    info "Binary already been replaced"
    exit 0
  fi

  K3S_CONTEXT=$(getfilecon $FULL_BIN_PATH 2>/dev/null | awk '{print $2}' || true)
  info "Deploying new k3s binary to $K3S_BIN_PATH"
  cp $NEW_BINARY $FULL_BIN_PATH

  if [ -n "${K3S_CONTEXT}" ]; then
    info 'Restoring k3s bin context'
    setfilecon "${K3S_CONTEXT}" $FULL_BIN_PATH
  fi

  info "K3s binary has been replaced successfully"
  return
}

kill_k3s_process() {
    # the script sends SIGTERM to the process and let the supervisor
    # to automatically restart k3s with the new version
    kill -SIGTERM $K3S_PID
    info "Successfully killed old k3s pid $K3S_PID"
}

prepare() {
  set +e
  KUBECTL_BIN="/opt/k3s kubectl"
  CONTROLPLANE_PLAN=${1}

  if [ -z "$CONTROLPLANE_PLAN" ]; then
    fatal "Control-plane Plan name was not passed to the prepare step. Exiting"
  fi

  NAMESPACE=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)
  while true; do
    # make sure control-plane plan does exist
    PLAN=$(${KUBECTL_BIN} get plan $CONTROLPLANE_PLAN -o jsonpath='{.metadata.name}' -n $NAMESPACE 2>/dev/null)
    if [ -z "$PLAN" ]; then
	    info "Waiting for control-plane Plan $CONTROLPLANE_PLAN to be created"
	    sleep 5
	    continue
    fi
    NUM_NODES=$(${KUBECTL_BIN} get plan $CONTROLPLANE_PLAN -n $NAMESPACE -o json | jq '.status.applying | length')
    if [ "$NUM_NODES" == "0" ]; then
      break
    fi
    info "Waiting for all control-plane nodes to be upgraded"
    sleep 5
  done
  verify_controlplane_versions
}

verify_controlplane_versions() {
  while true; do
    CONTROLPLANE_NODE_VERSION=$(${KUBECTL_BIN} get nodes --selector='node-role.kubernetes.io/control-plane' -o json | jq -r '.items[].status.nodeInfo.kubeletVersion' | sort -u | tr '+' '-')
    if [ -z "$CONTROLPLANE_NODE_VERSION" ]; then
      sleep 5
      continue
    fi
    if [ "$CONTROLPLANE_NODE_VERSION" == "$SYSTEM_UPGRADE_PLAN_LATEST_VERSION" ]; then
        info "All control-plane nodes have been upgraded to version to $CONTROLPLANE_NODE_VERSION"
		    break
		fi
    info "Waiting for all control-plane nodes to be upgraded to version $MODIFIED_VERSION"
	  sleep 5
	  continue
  done
}

upgrade() {
  get_k3s_process_info
  replace_binary
  kill_k3s_process
}

"$@"
```