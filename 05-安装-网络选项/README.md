Video Reference：[05-安装-网络选项](https://www.bilibili.com/video/BV1uR4y1M7Nm/?spm_id_from=333.788&vd_source=9560c118fae1db9638f05a6ba2527085)

GitHub README.md：[05-安装-网络选项](https://github.com/kingsd041/k3s-tutorial/tree/main/05-安装-网络选项)

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

| CLI Flag 和 Value           | 描述                                                         |
| --------------------------- | ------------------------------------------------------------ |
| --flannel-backend=vxlan     | (默认) 使用 VXLAN 后端。                                     |
| --flannel-backend=ipsec     | 使用 IPSEC 后端，对网络流量进行加密。                        |
| --flannel-backend=host-gw   | 使用 host-gw 后端。                                          |
| --flannel-backend=wireguard | 使用 WireGuard 后端，对网络流量进行加密。可能需要额外的内核模块和配置。 |
| --flannel-ipv6-masq         | 将伪装规则应用于 IPv6 流量（默认为 IPv4）。仅适用于 dual-stack 或仅 IPv6 集群 |

```shell
# 默认启动K3s集群
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -
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

```shell
# K3s master
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_EXEC="--flannel-backend=host-gw" \
  INSTALL_K3S_MIRROR=cn sh -
# 查看K3s的网络配置，以及路由配置
$ k3s kubectl get nodes
NAME         STATUS   ROLES                  AGE     VERSION
k8s-master   Ready    control-plane,master   2m16s   v1.25.6+k3s1
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.0.0.2        0.0.0.0         UG    102    0        0 ens33
10.0.0.0        0.0.0.0         255.255.255.0   U     102    0        0 ens33
10.42.0.0       0.0.0.0         255.255.255.0   U     0      0        0 cni0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
$ cat /var/lib/rancher/k3s/server/token
K10467b95f1899b85c1542e8c1769876d4783ff25a2f49cc362570e8fb79edbd44a::server:69a93dfd2992cd0e031dccc8043e704e

# K3s agent
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn K3S_URL=https://10.0.0.30:6443 \
  K3S_TOKEN=69a93dfd2992cd0e031dccc8043e704e sh -
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

$ $ k3s kubectl get nodes
NAME         STATUS   ROLES                  AGE     VERSION
k8s-master   Ready    control-plane,master   7m46s   v1.25.6+k3s1
k8s-node01   Ready    <none>                 36s     v1.25.6+k3s1

# K3s master
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.0.0.2        0.0.0.0         UG    102    0        0 ens33
10.0.0.0        0.0.0.0         255.255.255.0   U     102    0        0 ens33
10.42.0.0       0.0.0.0         255.255.255.0   U     0      0        0 cni0
10.42.1.0       10.0.0.31       255.255.255.0   UG    0      0        0 ens33

# K3s agent 
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.0.0.2        0.0.0.0         UG    102    0        0 ens33
10.0.0.0        0.0.0.0         255.255.255.0   U     102    0        0 ens33
10.42.0.0       10.0.0.30       255.255.255.0   UG    0      0        0 ens33
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
```

## 1.2 启用 Directrouting

当主机在同一子网时，启用 direct routes(如host-gw)。`vxlan` 只用于将数据包封装到不同子网的主机上，同子网的主机之间使用 `host-gw`。默认值为 `false`。

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

使用 `--flannel-backend=none` 运行 K3s，然后在安装你选择的 CNI。

## 2.1 Calico

Reference：https://docs.rancher.cn/docs/k3s/installation/network-options/_index

使用`--flannel-backend=none`运行 K3s，然后在安装你选择的 CNI。大多数 CNI 插件都有自己的网络策略引擎，因此建议同时设置 `--disable-network-policy` 以避免冲突。应该为 Canal 和 Calico 启用 IP 转发。请参考以下步骤。

- Canal
- Calico

按照Calico CNI 插件指南。修改 Calico YAML，在 `container_settings` 部分中允许 IP 转发，例如：

```shell
"container_settings": {
              "allow_ip_forwarding": true
          }
```

如不配置 `"allow_ip_forwarding": true`， `svclb-traefik` 将会报错：`/usr/bin/entry: line 6: can't create /proc/sys/net/ipv4/ip_forward: Read-only file system`

```shell
# K3s master
# --cluster-cidr 指定Pod的网络
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC="
  --docker 
  --flannel-backend=none \
  --cluster-cidr=192.168.0.0/16" \
  sh -

# 引用Calico网络插件
$ kubectl create -f https://raw.githubusercontent.com/kingsd041/k3s-tutorial/main/05-安装-网络选项/calico.yaml
$ kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
# 查看Node节点的情况
$ kubectl get nodes
NAME         STATUS   ROLES                  AGE     VERSION
k8s-master   Ready    control-plane,master   3m17s   v1.25.6+k3s1

# K3s agent
$ cat /var/lib/rancher/k3s/server/token
K108292174342c1e50b72900d10843d077bbcc5a05640ef8a167f4577179da6c1d7::server:2ccb74de1f46509f61ec0b425d4cace5

$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn INSTALL_K3S_EXEC="--docker" K3S_URL=https://10.0.0.30:6443 \
  K3S_TOKEN=2ccb74de1f46509f61ec0b425d4cace5 sh -

# 查看Node节点的情况
$ kubectl get nodes
NAME         STATUS   ROLES                  AGE     VERSION
k8s-node01   Ready    <none>                 2m30s   v1.25.6+k3s1
k8s-master   Ready    control-plane,master   25m     v1.25.6+k3s1
# 查看Pod的情况
$ kubectl get pod -A -o wide
NAMESPACE     NAME                                       READY   STATUS      RESTARTS   AGE     IP               NODE         NOMINATED NODE   READINESS GATES
kube-system   coredns-597584b69b-btm86                   1/1     Running     0          26m     172.16.235.194   k8s-master   <none>           <none>
kube-system   local-path-provisioner-79f67d76f8-bkm7l    1/1     Running     0          26m     172.16.235.195   k8s-master   <none>           <none>
kube-system   calico-node-nxhf7                          1/1     Running     0          24m     10.0.0.30        k8s-master   <none>           <none>
kube-system   svclb-traefik-ef72727b-b8kvv               2/2     Running     0          22m     172.16.235.199   k8s-master   <none>           <none>
kube-system   helm-install-traefik-crd-l4dbr             0/1     Completed   0          26m     172.16.235.197   k8s-master   <none>           <none>
kube-system   traefik-66c46d954f-t9r28                   1/1     Running     0          22m     172.16.235.200   k8s-master   <none>           <none>
kube-system   helm-install-traefik-qc769                 0/1     Completed   1          26m     172.16.235.196   k8s-master   <none>           <none>
kube-system   metrics-server-5f9f776df5-6r7sf            1/1     Running     0          26m     172.16.235.193   k8s-master   <none>           <none>
kube-system   calico-kube-controllers-74677b4c5f-lstln   1/1     Running     0          24m     172.16.235.198   k8s-master   <none>           <none>
kube-system   svclb-traefik-ef72727b-t5s29               2/2     Running     0          3m33s   172.16.85.193    k8s-node01   <none>           <none>
kube-system   calico-node-trnmm                          1/1     Running     0          3m54s   10.0.0.31        k8s-node01   <none>           <none>
```

参考：https://docs.projectcalico.org/getting-started/kubernetes/k3s/quickstart

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
    app: nginx-svc
  type: NodePort
  ports:
  - name: nginx-http
    protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
$ kubectl create -f nginx.yaml
$ kubectl get pod -l app=nginx-deploy -o wide
NAME                            READY   STATUS    RESTARTS   AGE   IP               NODE         NOMINATED NODE   READINESS GATES
nginx-deploy-65c9f9bbdf-9gxg4   1/1     Running   0          54s   172.16.235.211   k8s-master   <none>           <none>
nginx-deploy-65c9f9bbdf-9qwcb   1/1     Running   0          53s   172.16.235.213   k8s-master   <none>           <none>
nginx-deploy-65c9f9bbdf-xzczd   1/1     Running   0          54s   172.16.235.212   k8s-master   <none>           <none>
nginx-deploy-65c9f9bbdf-lpldm   1/1     Running   0          54s   172.16.235.214   k8s-master   <none>           <none>
nginx-deploy-65c9f9bbdf-gr759   1/1     Running   0          53s   172.16.235.215   k8s-master   <none>           <none>
nginx-deploy-65c9f9bbdf-c2k4q   1/1     Running   0          54s   172.16.85.194    k8s-node01   <none>           <none>
nginx-deploy-65c9f9bbdf-855gw   1/1     Running   0          54s   172.16.85.198    k8s-node01   <none>           <none>
nginx-deploy-65c9f9bbdf-crz4g   1/1     Running   0          54s   172.16.85.196    k8s-node01   <none>           <none>
nginx-deploy-65c9f9bbdf-ndstp   1/1     Running   0          54s   172.16.85.195    k8s-node01   <none>           <none>
nginx-deploy-65c9f9bbdf-9z4ml   1/1     Running   0          53s   172.16.85.197    k8s-node01   <none>           <none>
$ kubectl get svc nginx-svc
NAME        TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx-svc   NodePort   10.43.134.173   <none>        80:30080/TCP   3m27s

# 访问即可
```

## 2.2 Cilium

参考：https://docs.cilium.io/en/v1.9/gettingstarted/k3s/

## 2.3 macvlan

支持

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
3. 创建`macvlansubnet`导航到`项目`→`扁平网络`→`创建MacvlanSubnet` ;
4. 验证分别创建两个workload，并且启用扁平网络

ssh 到其中一个Pod内，然后ping另一个pod的`macvlan ip`：

# 3 只安装IPv6

K3s v1.22 或以上版本支持只安装 IPv6。

##### 💡提示：

如果你的 IPv6 默认路由是由路由器广告（RA）设置的，你需要设置 net.ipv6.conf.all.accept_ra = 2，否则，一旦默认路由过期，节点将放弃该路由。请注意，接受 RA 可能会增加[中间人攻击](https://github.com/kubernetes/kubernetes/issues/91507)的风险。