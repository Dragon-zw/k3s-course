Video Reference：[05-安装-网络选项](https://www.bilibili.com/video/BV1uR4y1M7Nm/?spm_id_from=333.788&vd_source=9560c118fae1db9638f05a6ba2527085)

GitHub README.md：[05-安装-网络选项](https://github.com/kingsd041/k3s-tutorial/tree/main/05-安装-网络选项)

**理解 Server 节点的安装，以及注册 Agent 节点的步骤！**

默认情况下，K3s 将以 flannel 作为 CNI 运行，使用 VXLAN 作为默认后端。CNI和默认后端都可以通过参数修改。

| **Flag**                        | **默认值**      | **描述**                                                    |
| ------------------------------- | --------------- | ----------------------------------------------------------- |
| --cluster-cidr value            | "10.42.0.0/16"  | 用于 Pod IP 的网络 CIDR                                     |
| --service-cidr value            | "10.43.0.0/16"  | 用于 service IP 的网络 CIDR                                 |
| --service-node-port-range value | "30000-32767"   | 为具有 NodePort 可见性的服务保留的端口范围                  |
| --cluster-dns value             | "10.43.0.10"    | 用于 coredns 服务的集群 IP。应该在您的service-cidr范围内    |
| --cluster-domain value          | "cluster.local" | 集群域名                                                    |
| --flannel-backend value         | "vxlan"         | 'none', 'vxlan', 'ipsec', 'host-gw', 或 'wireguard'中的一个 |

# 1 Flannel 选项

Flannel 的默认后端是 VXLAN。要启用加密，请使用下面的 IPSec（`Internet Protocol Security`）或 WireGuard 选项。

| **CLI Flag 和 Value**       | **描述**                                                     |
| --------------------------- | ------------------------------------------------------------ |
| --flannel-backend=vxlan     | (默认) 使用 VXLAN 后端。                                     |
| --flannel-backend=ipsec     | 使用 IPSEC 后端，对网络流量进行加密。                        |
| --flannel-backend=host-gw   | 使用 host-gw 后端。                                          |
| --flannel-backend=wireguard | 使用 WireGuard 后端，对网络流量进行加密。可能需要额外的内核模块和配置。 |
| --flannel-ipv6-masq         | 将伪装规则应用于 IPv6 流量（默认为 IPv4）。仅适用于 dual-stack 或仅 IPv6 集群 |

```shell
# 默认安装K3s集群之后的网络配置
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC="--docker" \
  K3S_KUBECONFIG_OUTPUT=/root/.kube/config \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN="rancher" sh -

$ ls /var/lib/rancher/k3s/agent/etc/
cni  containerd  crictl.yaml  flannel
$ cat /var/lib/rancher/k3s/agent/etc/flannel/net-conf.json
{
        "Network": "10.42.0.0/16",
        "EnableIPv6": false,
        "EnableIPv4": true,
        "IPv6Network": "::/0",
        "Backend": {
        "Type": "vxlan"
}
}
```

## 1.1 flannel-backend 使用 `host-gw`

原理就是将对方主机的IP地址，作为本机的默认网关，这样只要访问对方的Pod的话，就会把流量转发到对方的IP地址中。

这样，我就可以在安装 `K3s` 或者之后修改对应配置文件，来修改 `Flannel` 默认的后端网络配置选项(重启会覆盖不生效)了。下面，我们演示下，如何修改为 `host-gw` 模式。

```shell
# K3s master
# 主节点
# flannel-backend使用host-gw
# 该模式会把对端主机的IP当做默认网管(多Server情况)
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC="--flannel-backend=host-gw --docker" \
  K3S_KUBECONFIG_OUTPUT=/root/.kube/config \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN="rancher" sh -

# 查看K3s的网络配置，以及路由配置
$ k3s kubectl get nodes
NAME   STATUS   ROLES                  AGE   VERSION
k3s1   Ready    control-plane,master   36s   v1.21.14+k3s1

# 默认的路由信息
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.0.0.2        0.0.0.0         UG    102    0        0 ens33
10.0.0.0        0.0.0.0         255.255.255.0   U     102    0        0 ens33
10.42.0.0       0.0.0.0         255.255.255.0   U     0      0        0 cni0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0

$ cat /var/lib/rancher/k3s/server/token
K10467b95f1899b85c1542e8c1769876d4783ff25a2f49cc362570e8fb79edbd44a::server:rancher

# K3s agent
# 工作节点
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn K3S_URL=https://10.0.0.51:6443 \
  INSTALL_K3S_EXEC="--docker" \
  K3S_KUBECONFIG_OUTPUT=/root/.kube/config \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN=rancher sh -
# 查看配置之后的网络配置
$ cat /var/lib/rancher/k3s/agent/etc/flannel/net-conf.json
{
        "Network": "10.42.0.0/16",
        "EnableIPv6": false,
        "EnableIPv4": true,
        "IPv6Network": "::/0",
        "Backend": {
        "Type": "host-gw"
}
}

$ k3s kubectl get nodes
NAME   STATUS   ROLES                  AGE   VERSION
k3s1   Ready    control-plane,master   69s   v1.21.14+k3s1
k3s2   Ready    <none>                 14s   v1.21.14+k3s1

# K3s master
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.0.0.2        0.0.0.0         UG    102    0        0 ens33
10.0.0.0        0.0.0.0         255.255.255.0   U     102    0        0 ens33
10.42.0.0       0.0.0.0         255.255.255.0   U     0      0        0 cni0
10.42.1.0       10.0.0.31       255.255.255.0   UG    0      0        0 ens33
# 默认会将10.42.1.0/24的流量都转发到10.0.0.31默认网关(K3s Agent的IP地址)

# K3s agent 
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.0.0.2        0.0.0.0         UG    102    0        0 ens33
10.0.0.0        0.0.0.0         255.255.255.0   U     102    0        0 ens33
10.42.0.0       10.0.0.30       255.255.255.0   UG    0      0        0 ens33
# 默认会将10.42.0.0/24的流量都转发到10.0.0.30默认网关(K3s Server的IP地址)
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
```

## 1.2 启用 Directrouting

当主机在同一子网时，启用 direct routes`(如host-gw)`。`vxlan` 只用于将数据包封装到不同子网的主机上，同子网的主机之间使用 `host-gw`。默认值为 `false`。

要添加我们就不能修改其对应的网络配置文件，因为重新安装或者重启都会把这个配置冲掉(变成默认配置)，所以需要折中下。我们自建一个网络配置文件，然后在启动的时候执行从哪个配置文件里面加载对应配置。

```shell
# K3s master 和 agent
cat >> /etc/net-conf.json <<-'EOF' 
{
        "Network": "10.42.0.0/16",
        "EnableIPv6": false,
        "EnableIPv4": true,
        "IPv6Network": "::/0",
        "Backend": {
        "Type": "vxlan"
        "Directrouting": true
}
}
EOF

# K3s master
# --flannel-conf 用来自定义flannel的配置文件
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_EXEC="--flannel-conf=/etc/net-conf.json" \
  INSTALL_K3S_MIRROR=cn sh -

# 查看路由信息
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.0.0.2        0.0.0.0         UG    104    0        0 ens33
10.0.0.0        0.0.0.0         255.255.255.0   U     104    0        0 ens33
10.42.1.0       10.0.0.31       255.255.255.0   UG    0      0        0 ens33
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
```

# 2 自定义 CNI

使用 `--flannel-backend=none` 运行 K3s，然后在安装你选择的 CNI。按照 [Calico CNI 插件指南](https://docs.projectcalico.org/master/reference/cni-plugin/configuration) 来修改 Calico 的 YAML 配置文件，在 container_settings 部分中允许 IP 转发。

## 2.1 Calico

Reference：https://docs.rancher.cn/docs/k3s/installation/network-options/_index

使用`--flannel-backend=none`运行 K3s，然后在安装你选择的 CNI。大多数 CNI 插件都有自己的网络策略引擎，因此建议同时设置 `--disable-network-policy` 以避免冲突。应该为 Canal 和 Calico 启用 IP 转发。请参考以下步骤。

- Canal
- Calico

按照Calico CNI 插件指南。修改 Calico YAML，在 `container_settings` 部分中允许 IP 转发，例如：

```shell
# 加到Calico的YAML文件中
# 允许IP转发(这个是K3s的一个限制；需要开启)
"container_settings": {
    "allow_ip_forwarding": true
}

- name: CALICO_IPV4POOL_CIDR
  value: "192.168.200.0/24"

# 通过在主机上运行以下命令，确保设置已被应用(true)
$ sudo cat /etc/cni/net.d/10-canal.conflist
```

如不配置 `"allow_ip_forwarding": true`， `svclb-traefik` 将会报错：`/usr/bin/entry: line 6: can't create /proc/sys/net/ipv4/ip_forward: Read-only file system`

```shell
# K3s master
# --cluster-cidr 指定Pod的网络(其中--cluster-cidr可不设置)
# --flannel-backend=none 主要的目的是启动k3s集群的时候，禁用掉默认的flanneld的网络插件
# 即没有任何CNI的网络插件的
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC="
  --docker 
  --flannel-backend=none \
  --cluster-cidr=192.168.200.0/16" \
  K3S_KUBECONFIG_OUTPUT=/root/.kube/config \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN="rancher" sh -

# 引用Calico网络插件(启动网络服务).
# kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
$ kubectl create -f https://raw.githubusercontent.com/kingsd041/k3s-tutorial/main/05-安装-网络选项/calico.yaml

# 查看Node节点的情况
$ kubectl get nodes
NAME         STATUS   ROLES                  AGE     VERSION
k3s1         Ready    control-plane,master   3m17s   v1.21.14+k3s1
# 查看K3s Server的Token信息
$ cat /var/lib/rancher/k3s/server/token
K108292174342c1e50b72900d10843d077bbcc5a05640ef8a167f4577179da6c1d7::server:rancher

#############################################################################################################
# K3s agent
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC="--docker" \
  K3S_URL=https://10.0.0.51:6443 \
  K3S_KUBECONFIG_OUTPUT=/root/.kube/config \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN=rancher sh -

# 查看Node节点的情况
$ kubectl get nodes
NAME   STATUS   ROLES                  AGE     VERSION
k3s1   Ready    control-plane,master   5m58s   v1.21.14+k3s1
k3s2   Ready    <none>                 103s    v1.21.14+k3s1

# 查看Calico Pod的情况
$ kubectl get pod -A -o wide | grep calico
kube-system   calico-node-z6b74                         1/1     Running     0          4m14s   10.0.0.51         k3s1   <none>           <none>
kube-system   calico-kube-controllers-b656ddcfc-2jz8c   1/1     Running     0          4m13s   192.168.200.132   k3s1   <none>           <none>
kube-system   calico-node-nn24j                         1/1     Running     0          2m36s   10.0.0.52         k3s2   <none>           <none
```

参考资料：https://docs.projectcalico.org/getting-started/kubernetes/k3s/quickstart

部署 Calico 网络插件：kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml 

[ 推荐使用`Manifest`的方式部署 Calico ]

测试集群Yaml文件

```yaml
---
# nginx-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name:  nginx-deploy
  namespace: default
  labels:
    app:  nginx-deploy
spec:
  selector:
    matchLabels:
      app: nginx-deploy
  replicas: 10
  template:
    metadata:
      labels:
        app:  nginx-deploy
    spec:
      containers:
      - name:  nginx-deploy
        image:  nginx:1.23.3-alpine
        ports:
        - containerPort:  80
          name:  http
---
# nginx-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: default
spec:
  selector:
    app: nginx-deploy
  type: NodePort
  ports:
  - name: nginx-http
    protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
$ kubectl create -f nginx-deploy.yaml
# 查看Pod的状态(以及对应的Pod的IP地址)
$ kubectl get pod -l app=nginx-deploy -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP                NODE   NOMINATED NODE   READINESS GATES
nginx-deploy-d4c47bf76-7njtz   1/1     Running   0          85s   192.168.200.194   k3s2   <none>           <none>
nginx-deploy-d4c47bf76-tr7vq   1/1     Running   0          85s   192.168.200.137   k3s1   <none>           <none>
nginx-deploy-d4c47bf76-79txg   1/1     Running   0          85s   192.168.200.195   k3s2   <none>           <none>
nginx-deploy-d4c47bf76-qqd6p   1/1     Running   0          85s   192.168.200.138   k3s1   <none>           <none>
nginx-deploy-d4c47bf76-mf8dr   1/1     Running   0          85s   192.168.200.196   k3s2   <none>           <none>
nginx-deploy-d4c47bf76-h4l5g   1/1     Running   0          85s   192.168.200.139   k3s1   <none>           <none>
nginx-deploy-d4c47bf76-wbnxz   1/1     Running   0          85s   192.168.200.197   k3s2   <none>           <none>
nginx-deploy-d4c47bf76-x55fc   1/1     Running   0          85s   192.168.200.140   k3s1   <none>           <none>
nginx-deploy-d4c47bf76-j65b7   1/1     Running   0          85s   192.168.200.198   k3s2   <none>           <none>
nginx-deploy-d4c47bf76-4wsmw   1/1     Running   0          85s   192.168.200.141   k3s1   <none>           <none>
# 查看SVC的服务地址
$ kubectl get svc nginx-svc
NAME        TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-svc   NodePort   10.43.230.18   <none>        80:30080/TCP   90s
$ kubectl describe endpoints nginx-svc
Name:         nginx-svc
Namespace:    default
Labels:       <none>
Annotations:  endpoints.kubernetes.io/last-change-trigger-time: 2023-03-26T05:11:02+08:00
Subsets:
  Addresses:          192.168.200.137,192.168.200.138,192.168.200.139,192.168.200.140,192.168.200.141,192.168.200.194,192.168.200.195,192.168.200.196,192.168.200.197,192.168.200.198
  NotReadyAddresses:  <none>
  Ports:
    Name        Port  Protocol
    ----        ----  --------
    nginx-http  80    TCP

Events:  <none>

# 访问即可
```

## 2.2 Cilium

参考：https://docs.cilium.io/en/v1.9/gettingstarted/k3s/

GitHub 资料：https://github.com/cilium/cilium

Cilium 是一种网络、可观察性和安全性解决方案，具有基于 eBPF 的 数据平面。它提供了一个简单的扁平第 3 层网络，能够跨越 本地路由或覆盖模式下的多个集群。它是L7协议 感知并可以使用基于身份的安全性在 L3-L7 上实施网络策略 与网络寻址分离的模型。

Cilium 为 Pod 之间的流量实现了分布式负载平衡，并且 外部服务，并且能够完全取代 kube 代理，使用高效的 eBPF 中的哈希表允许几乎无限的扩展。它还支持 高级功能，如集成入口和出口网关、带宽 管理和服务网格化，并提供深入的网络和安全可见性和监控。

一种新的Linux内核技术称为[eBPF](https://ebpf.io/)，是Cilium的基础。它 支持将 eBPF 字节码动态插入 Linux 内核 集成点，例如：网络 IO、应用程序套接字和跟踪点 实施安全性、网络和可见性逻辑。eBPF非常高效 和灵活性。要了解有关 eBPF 的更多信息，请访问 [eBPF.io](https://ebpf.io/)。

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679779003914-25f4c92f-311c-4ad0-bd15-46e94f4b7bb0.png)

Cilium 是一个用于容器网络领域的开源项目，主要是面向容器而使用，用于提供并透明地保护应用程序工作负载（如应用程序容器或进程）之间的网络连接和负载均衡。

Cilium 在第 3/4 层运行，以提供传统的网络和安全服务，还在第 7 层运行，以保护现代应用协议（如 HTTP, gRPC 和 Kafka）的使用。 Cilium 被集成到常见的容器编排框架中，如 Kubernetes 和 Mesos。

Cilium 的底层基础是 BPF，Cilium 的工作模式是生成内核级别的 BPF 程序与容器直接交互。区别于为容器创建 overlay 网络，Cilium 允许每个容器分配一个 IPv6 地址（或者 IPv4 地址），使用容器标签而不是网络路由规则去完成容器间的网络隔离。它还包含创建并实施 Cilium 规则的编排系统的整合。

## 2.3 macvlan

支持。准确说这是一种网卡虚拟化的解决方案。因为 macvlan 这种技术能将 `**一块物理网卡虚拟成多块虚拟网卡**` ，相当于物理网卡施展了 多重影分身之术 ，由一个变多个。

### 2.3.1 测试环境

- 两台阿里云主机，1台安装k3s集群，1台安装Rancher企业版
- k3s: v1.18.8+k3s1

### 2.3.2 安装步骤

1. 启动单节点K3S集群，并禁用`flannel`

```shell
$ curl -sfl https: //docs.rancher.cn/k3s/k3s-install.sh |INSTALL_K3S_MIROR=cn \
  INSTALL_K3S_EXEC="--no-flannel" sh -
```

1. 部署`macvlan`

```shell
wget https://github.com/kingsd041/network-controller/releases/dowumload/v.5.6/multus-flannel-macvlan.yaml

kubectl create -f multus-flannel-macvlan.yaml
```

确认Pod是否都启动成功

```shell
kubectl get pods -A
```

1. 在另一台主机上安装rancher企业版
2. 将K3s集群导入到Rancher企业版
3. 创建`macvlansubnet`导航到`项目`→`扁平网络`→`创建MacvlanSubnet`
4. 验证分别创建两个workload，并且启用扁平网络

ssh 到其中一个Pod内，然后ping另一个pod的`macvlan ip`

# 3 只安装IPv6

K3s v1.22 或以上版本支持只安装 IPv6。

##### 💡提示：

如果你的 IPv6 默认路由是由路由器广告（RA）设置的，你需要设置 net.ipv6.conf.all.accept_ra = 2，否则，一旦默认路由过期，节点将放弃该路由。请注意，接受 RA 可能会增加[中间人攻击](https://github.com/kubernetes/kubernetes/issues/91507)的风险。