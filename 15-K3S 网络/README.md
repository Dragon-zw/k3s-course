Video Reference：[15-网络](https://www.bilibili.com/video/BV1tZ4y1f7Bb/?spm_id_from=333.788&vd_source=9560c118fae1db9638f05a6ba2527085)

GitHub README.md：[15-网络](https://github.com/kingsd041/k3s-tutorial/tree/main/15-网络)

官方文档资料：https://docs.k3s.io/zh/networking

# 1 网络

本节课介绍了 `CoreDNS`、`Traefik Ingress` 控制器和 `Klipper service load balancer` 如何在 K3s 中工作。

关于 Flannel 配置选项和后端选择，或如何设置自己的 CNI，请参考 [安装网络选项](http://docs.rancher.cn/docs/k3s/installation/network-options/_index) 页面。

关于 K3s 需要开放哪些端口，请参考 [安装要求](http://docs.rancher.cn/docs/k3s/installation/installation-requirements/_index#网络)。

K3s server 需要 6443 端口才能被所有节点访问。

当使用 Flannel VXLAN 时，节点需要能够通过 UDP 端口 8472 访问其他节点，或者当使用 Flannel Wireguard 后端时，节点需要能够通过 UDP 端口 51820 和 51821（使用 IPv6 时）访问其他节点。该节点不应侦听任何其他端口。 K3s 使用反向隧道，以便节点与服务器建立出站连接，并且所有 kubelet 流量都通过该隧道运行。但是，如果你不使用 Flannel 并提供自己的自定义 CNI，那么 K3s 不需要 Flannel 所需的端口。

如果要使用metrics server，则需要在每个节点上打开端口 10250 端口。

如果计划使用嵌入式 etcd 实现高可用性，则 server 节点必须在端口 2379 和 2380 上可以相互访问。

##### 💡重要

节点上的 VXLAN 端口不应公开暴露，因为它公开了集群网络，任何人都可以访问它。应在禁止访问端口 8472 的防火墙/安全组后面运行节点。

**⚠️警告：** Flannel 依靠 [Bridge CNI plugin](https://www.cni.dev/plugins/current/main/bridge/) 来创建一个可以交换流量的 L2 网络。具有 NET_RAW 功能的 Rogue pod 可以滥用该 L2 网络来发动攻击，如 [ARP 欺骗](https://static.sched.com/hosted_files/kccncna19/72/ARP DNS spoof.pdf)。因此，正如 [kubernetes 文档](https://kubernetes.io/docs/concepts/security/pod-security-standards/)中记载的那样，请设置一个受限配置文件，在不可信任的 pod 上禁用 NET_RAW。

| **协议** | **端口**  | **源**                   | **描述**                                      |
| -------- | --------- | ------------------------ | --------------------------------------------- |
| TCP      | 6443      | K3s agent 节点           | Kubernetes API Server                         |
| UDP      | 8472      | K3s server 和 agent 节点 | 仅对 Flannel VXLAN 需要                       |
| UDP      | 51820     | K3s server 和 agent 节点 | 只有 Flannel Wireguard 后端需要               |
| UDP      | 51821     | K3s server 和 agent 节点 | 只有使用 IPv6 的 Flannel Wireguard 后端才需要 |
| TCP      | 10250     | K3s server 和 agent 节点 | Kubelet metrics                               |
| TCP      | 2379-2380 | K3s server 节点          | 只有嵌入式 etcd 高可用才需要                  |

## 1.1 💎CoreDNS

CoreDNS 是在 agent 节点启动时部署的。要禁用，请在每台服务器上运行 `--disable coredns` 选项。

如果你不安装 CoreDNS，你将需要自己安装一个集群 DNS 提供商[ 需要手动部署 CoreDNS，一般实际使用并不会禁用 CoreDNS 功能 ]。

### 1.1.1 如何修改 coredns 参数

通过修改`/var/lib/rancher/k3s/server/manifests/coredns.yaml`配置文件会立即生效，但重启 K3s 服务会导致 coredns 配置重新初始化，所以要修改 corends 的参数，可以通过以下步骤修改：

1. 将`coredns.yaml`保存到其他目录
2. 通过 `--disable coredns` 禁用 `coredns`
3. 将备份的`coredns.yaml` 复制到 `/var/lib/rancher/k3s/server/manifests/c.yaml`，并修改对应参数

```shell
# K3s Master
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC="--docker" \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN="rancher" sh -

# 查看Node状态
$ kubectl get nodes
NAME   STATUS   ROLES                  AGE   VERSION
k3s1   Ready    control-plane,master   28s   v1.21.14+k3s1

# 查看Token信息
$ cat /var/lib/rancher/k3s/server/token
K10436aaa99ab2808a8a33f8737e0aea5bad9ed3a3afbababb11e68e982261f065a::server:rancher

# K3s Worker
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  K3S_URL=https://10.0.0.51:6443 \
  K3S_TOKEN=rancher \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  INSTALL_K3S_EXEC="--node-ip=10.0.0.52" \
  sh -

# 查看Node状态
$ kubectl get nodes
NAME   STATUS   ROLES                  AGE     VERSION
k3s1   Ready    control-plane,master   3m49s   v1.21.14+k3s1
k3s2   Ready    <none>                 28s     v1.21.14+k3s1

# 修改coredns.yaml文件
$ cd /var/lib/rancher/k3s/server/manifests
# cp -av /var/lib/rancher/k3s/server/manifests/coredns.yaml /var/lib/rancher/k3s/server/manifests/c.yaml
$ vim coredns.yaml
[......]
Corefile: |
    .:53 {
        hosts /etc/coredns/NodeHosts {
          ttl 60
          reload 15s
          10.0.0.51 k3s.test.org
          fallthrough
        }
[......]

# 保存文件之后，就会类似于 kubectl apply -f coredns.yaml 重新引用该配置文件
# kubectl get configmap coredns -n kube-system -o yaml
```

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679577260163-62184ba1-ca2e-4451-a0e5-911059dc5c5b.png)

范例：创建一个 Nginx Pod 进行测试

```shell
# 下载镜像(Containerd)
$ ctr images pull --all-platforms docker.io/kingsd/nginx:install-tools
# 下载镜像(Docker)
$ docker pull docker.io/kingsd/nginx:install-tools

# 运行Pod并查看Pod的状态
$ kubectl run nginx --image=kingsd/nginx:install-tools
$ kubectl get pod -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP          NODE   NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          69s   10.42.1.3   k3s2   <none>           <none>

$ kubectl exec -it nginx -- ping -c 1 -W 1 k3s.test.org
PING k3s.test.org (10.0.0.51) 56(84) bytes of data.
64 bytes from master01 (10.0.0.51): icmp_seq=1 ttl=63 time=2.69 ms

--- k3s.test.org ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 2.692/2.692/2.692/0.000 ms
```

- 当重启 k3s 集群时，会将修改后的`coredns.yaml`配置文件进行重新初始化成系统默认的配置。

```shell
# 重启k3s服务
$ systemctl restart k3s.service
# 重新查看之后就变成默认配置
$ cat /var/lib/rancher/k3s/server/manifests/coredns.yaml

# 为了方便后续的实验，将该coredns.yaml拷贝到/tmp下
$ cp -av /var/lib/rancher/k3s/server/manifests/coredns.yaml /tmp/cdns.yaml
```

- 正确的做法

```shell
# 禁用coredns的功能
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_EXEC="--disable coredns --docker" \
  INSTALL_K3S_VERSION="v1.21.14+k3s1" \
  K3S_TOKEN="rancher" sh -

# 会发现已经没有该coredns.yaml配置文件
$ ls -l /var/lib/rancher/k3s/server/manifests
total 16
-rw------- 1 root root 1108 Mar 23 21:26 ccm.yaml
-rw------- 1 root root 3645 Mar 23 21:26 local-storage.yaml
drwx------ 2 root root  227 Mar 23 18:29 metrics-server
-rw------- 1 root root 1039 Mar 23 21:26 rolebindings.yaml
-rw------- 1 root root 1098 Mar 23 21:26 traefik.yaml

# 可以查看Pod的运行情况
$ kubectl get pod --all-namespaces
# 编辑 cdns.yaml 文件
# 类似于手动部署 CoreDNS 的方式
$ vim /tmp/cdns.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
- apiGroups:
  - discovery.k8s.io
  resources:
  - endpointslices
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        hosts /etc/coredns/NodeHosts {
          ttl 60
          reload 15s
          10.0.0.51 k3s.test.org
          fallthrough
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
  NodeHosts: |
    10.0.0.51 k3s1
    10.0.0.52 k3s2
# 需要注意添加 NodeHosts否则部署DNS会失败
# 其实NodeHosts在启动K3s集群之后，会默认生成key，因为这就是对应IP地址和对应的主机名
# 不手动创建NodeHosts key的话会创建失败，原因就是找到该NodeHosts key
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/name: "CoreDNS"
spec:
  #replicas: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
    spec:
      priorityClassName: "system-cluster-critical"
      serviceAccountName: coredns
      tolerations:
        - key: "CriticalAddonsOnly"
          operator: "Exists"
        - key: "node-role.kubernetes.io/control-plane"
          operator: "Exists"
          effect: "NoSchedule"
        - key: "node-role.kubernetes.io/master"
          operator: "Exists"
          effect: "NoSchedule"
      nodeSelector:
        beta.kubernetes.io/os: linux
      containers:
      - name: coredns
        image: rancher/mirrored-coredns-coredns:1.9.1
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
          readOnly: true
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 1
          successThreshold: 1
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8181
            scheme: HTTP
          initialDelaySeconds: 0
          periodSeconds: 2
          timeoutSeconds: 1
          successThreshold: 1
          failureThreshold: 3
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
            - key: NodeHosts
              path: NodeHosts
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.43.0.10
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
  - name: metrics
    port: 9153
    protocol: TCP
# 注意不能将c_dns.yaml修改为coredns.yaml文件，否则K3s会认为是系统初始化的默认配置文件
# 由于是"--disable coredns"，所以若命名为coredns.yaml，K3s重启服务后会删除该文件
$ cp -av /tmp/cdns.yaml /var/lib/rancher/k3s/server/manifests/
$ ls -l /var/lib/rancher/k3s/server/manifests/
total 24
-rw------- 1 root root 1108 Mar 23 21:26 ccm.yaml
-rw------- 1 root root 4483 Mar 23 21:31 cdns.yaml
-rw------- 1 root root 3645 Mar 23 21:26 local-storage.yaml
drwx------ 2 root root  227 Mar 23 18:29 metrics-server
-rw------- 1 root root 1039 Mar 23 21:26 rolebindings.yaml
-rw------- 1 root root 1098 Mar 23 21:26 traefik.yaml

# 查看是否已经创建好coredns的Pod
$ kubectl get pod --all-namespaces | grep coredns
kube-system   coredns-574bcc6c46-znlzf                  1/1     Running     0          5m17s

# 进行测试
$ kubectl run nginx --image=kingsd/nginx:install-tools
$ kubectl get pod -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP          NODE   NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          17m   10.42.0.9   k3s1   <none>           <none>

$ kubectl exec -it nginx -- ping -c 1 -W 1 k3s.test.org
PING k3s.test.org (10.0.0.51) 56(84) bytes of data.
64 bytes from ip-10-0-0-51 (10.0.0.51): icmp_seq=1 ttl=63 time=1.14 ms

--- k3s.test.org ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.147/1.147/1.147/0.000 m

# 重启K3s集群服务，查看配置是否会初始化配置
$ systemctl restart k3s.service
# 会发现并不会对coredns的配置做修改
$ cat /var/lib/rancher/k3s/server/manifests/cdns.yaml
```

## 1.2 💎Traefik Ingress Controller

在 K3s 中，内置了 Traefik 作为集群的默认反向代理和 Ingress Controller。K3s 1.21 开始默认安装 Traefik v2，而之前的版本则默认安装 Traefik v1。本文将根据不同的 Traefik 版本来介绍如何启用 Traefik Dashborad。

Traefik 是一个云原生的新型的 HTTP 反向代理、[负载均衡](https://cloud.tencent.com/product/clb?from=10680)软件，能轻易的部署微服务。k3s 将其作为集群的默认反向代理 和 Ingress Controller，但可视化面板是无法访问的。

启动 server 时，默认情况下会部署 Traefik。默认的配置文件在`/var/lib/rancher/k3s/server/manifests/traefik.yaml`中，对该文件的任何修改都会以类似`kubectl apply`的方式自动部署到 Kubernetes 中。Traefik ingress controller 将使用主机上的 `80` 和 `443` 端口（即这些端口不能用于 HostPort 或 NodePort）。

Traefik 可以通过编辑`traefik.yaml`文件进行配置。为了防止 k3s 使用或覆盖修改后的版本，请使用`--disable traefik`部署 k3s，并将修改后的副本存储在`k3s/server/manifests`目录中。更多信息请参考官方的 [Traefik 配置参数](https://github.com/helm/charts/tree/master/stable/traefik#configuration)。

⚠️要禁用 Traefik，请使用`--disable traefik`选项启动每个 server。

```shell
# 会发现并不会有进程监听80和443端口(Traefik Ingress Controller)
$ netstat -auntlp | grep -E "80|443"
# 主要是因为Traefik没有把80和443监听到主机上，但是并不会影响其功能
# Traefik主要是通过iptables的方式进行转发

# 查看iptables规则
$ iptables -nL -t nat | grep "\<443\>"
CNI-HOSTPORT-SETMARK  tcp  --  10.42.0.0/24         0.0.0.0/0            tcp dpt:443
CNI-HOSTPORT-SETMARK  tcp  --  127.0.0.1            0.0.0.0/0            tcp dpt:443
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:443 to:10.42.0.7:443
CNI-DN-db7ad52b865fb5d94c977  tcp  --  0.0.0.0/0            0.0.0.0/0            /* dnat name: "cbr0" id: "deb7a2b32ae653e362388b98e974787ecb01bea252713f39606707b4980bb850" */ multiport dports 80,443
KUBE-MARK-MASQ  tcp  -- !10.42.0.0/16         10.43.0.1            /* default/kubernetes:https cluster IP */ tcp dpt:443
KUBE-SVC-NPX46M4PTMTKRN6Y  tcp  --  0.0.0.0/0            10.43.0.1            /* default/kubernetes:https cluster IP */ tcp dpt:443
KUBE-MARK-MASQ  tcp  -- !10.42.0.0/16         10.43.205.86         /* kube-system/metrics-server:https cluster IP */ tcp dpt:443
KUBE-SVC-Z4ANX4WAEWEBLCTM  tcp  --  0.0.0.0/0            10.43.205.86         /* kube-system/metrics-server:https cluster IP */ tcp dpt:443
KUBE-MARK-MASQ  tcp  -- !10.42.0.0/16         10.43.154.214        /* kube-system/traefik:websecure cluster IP */ tcp dpt:443
KUBE-SVC-CVG3OEGEH7H5P3HQ  tcp  --  0.0.0.0/0            10.43.154.214        /* kube-system/traefik:websecure cluster IP */ tcp dpt:443
KUBE-FW-CVG3OEGEH7H5P3HQ  tcp  --  0.0.0.0/0            10.0.0.51            /* kube-system/traefik:websecure loadbalancer IP */ tcp dpt:443
KUBE-FW-CVG3OEGEH7H5P3HQ  tcp  --  0.0.0.0/0            10.0.0.52            /* kube-system/traefik:websecure loadbalancer IP */ tcp dpt:44

$ kubectl get pod -o wide -A | grep "10.42.0.7"
kube-system   svclb-traefik-v792h                       2/2     Running     0          3h13m   10.42.0.7    master01   <none>           <none>

$ kubectl exec -it -n kube-system svclb-traefik-v792h -- /bin/sh
Defaulted container "lb-port-80" out of: lb-port-80, lb-port-443
/ # iptables -L -t nat | grep 443
DNAT       tcp  -- !traefik.kube-system.svc.cluster.local  anywhere             tcp dpt:https to:10.43.154.214:443
# 将Pod的流量都转发到了10.43.154.214:443地址中

# 可以查看到该10.43.154.214是一个Service的地址
$ kubectl get svc -A | grep 10.43.154.214
kube-system   traefik          LoadBalancer   10.43.154.214   10.0.0.51,10.0.0.52   80:32263/TCP,443:30906/TCP   3h14m
# 总结：Traefik通过这一系列的转发，由Traefik来处理下一跳的流量
```

### 1.2.1 Traefik v1 启用 Dashborad

默认情况下，K3s 1.20 及更早版本默认安装 Traefik v1，并且默认没有启用 Traefik Dashboard。如果要在 K3s 中启用 Traefik v1 的 Dashborad，我们可以借助 HelmChartConfig 来自定义由 Helm 部署的 Traefik v1 并启用 Dashboard：

注意：

- 不建议手动编辑 `/var/lib/rancher/K3s/server/manifests/traefik.yaml` 来修改 Traefik 配置文件，因为 K3s 重启后会覆盖修改的内容。
- 建议通过在 `/var/lib/rancher/K3s/server/manifests` 中创建一个额外的 `HelmChartConfig`清单来自定义 Traefik 配置，请参考：`http://docs.rancher.cn/docs/K3s/helm/_index/`

```yaml
cat >> /var/lib/rancher/K3s/server/manifests/traefik-config.yaml << EOF
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    dashboard:
      enabled: true
      domain: "traefik.localhost"
EOF
```

此时，Traefik 将会重新部署，大约 10 秒钟左右，就可以通过 `**spec.valuesContent.domain**` 配置的域名来访问 Traefik Dashboard 了:

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679844609503-4b54aacf-da8b-4357-a19f-96717f62cdcc.png)

### 1.2.2 如何启用 Treafik2 dashboard（可选）：

默认情况下，K3s 1.21 及更高版本默认安装 Traefik v2。出于安全考虑，默认是不公开 Traefik Dashboard 的。

```yaml
# treafik2_dashboard.yaml
# Note: in a kubernetes secret the string (e.g. generated by htpasswd) must be base64-encoded first.
# To create an encoded user:password pair, the following command can be used:
# htpasswd -nb admin      admin      | openssl base64
# htpasswd -nb [username] [password] | openssl base64

apiVersion: v1
kind: Secret
metadata:
  name: authsecret
  namespace: default
data:
  users: |2
    YWRtaW46JGFwcjEkLkUweHd1Z0EkUjBmLi85WndJNXZWRFMyR2F2LmtELwoK
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
spec:
  entryPoints:
    - traefik
  routes:
  - match: Host(`traefik.example.com`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))
    kind: Rule
    services:
    - name: api@internal
      kind: TraefikService
    middlewares:
      - name: auth
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: auth
spec:
  basicAuth:
    secret: authsecret # Kubernetes secret named "secretName"
$ kubectl create -f treafik2_dashboard.yaml
secret/authsecret created
ingressroute.traefik.containo.us/traefik-dashboard created
middleware.traefik.containo.us/auth created

# 查看本机的/etc/hosts文件
# 添加相应的条目
10.0.0.51 traefik.example.com
```

然后通过 http://traefik.example.com/dashboard/ 访问 traefik dashboard

👑要访问 Traefik 仪表板，请浏览到以下 URL。注意`**尾部斜杠**`很重要且需要，否则您将收到错误404。

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679846023231-6621e7ff-8deb-4231-bac0-a600a5e55587.png)

### 1.2.3 👑Traefik v2 启用 Dashborad

默认情况下，K3s 1.21 及更高版本默认安装 Traefik v2。出于安全考虑，默认是不公开 Traefik Dashboard 的。我们常见的公开 Dashborad 的方式主要为以下两种：

#### 1.2.3.1 **方法 1：通过端口转发来实现**

```shell
kubectl port-forward  -n kube-system $(kubectl get pods -n kube-system| grep '^traefik-' | awk '{print $1}') --address 0.0.0.0 9000:9000
```

端口转发开启后，可以通过 `http://127.0.0.1:9000/dashboard/` 来访问 Dashboard：

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679846024912-1fc2c368-17f9-4d86-8ab8-53eeb94a5fc9.png)

#### 1.2.3.2 **方法 2：自定义 IngressRoute CRD**

另一种方法是通过定义和应用 IngressRoute CRD (`kubectl apply -f dashboard.yaml`)：

```yaml
# dashboard.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: dashboard
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`traefik.example`) && (PathPrefix(`/dashboard`) || PathPrefix(`/api`))
      kind: Rule
      services:
      - name: api@internal
        kind: TraefikService
```

部署成功后，可通过 http://traefik.example/dashboard/ 访问 Dashboard：

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679846026668-4192e851-388c-40ff-871b-df6d20db1944.png)

参考资料：

https://doc.traefik.io/traefik/operations/dashboard/

https://doc.traefik.io/traefik/middlewares/basicauth/#general

https://www.jianshu.com/p/5e576b4545fc

## 1.3 💎Service Load Balancer

K3s 提供了一个名为 [Klipper Load Balancer](https://github.com/rancher/klipper-lb) 的负载均衡器，它可以使用可用的主机端口。 允许创建 LoadBalancer 类型的 Service，但不包括 LB 的实现。某些 LB 服务需要云提供商，例如 Amazon EC2 或 Microsoft Azure。相比之下，`K3s service LB 使得可以在没有云提供商的情况下使用 LB 服务`。

### 1.3.1 Service LB 如何工作

K3s 创建了一个控制器，该控制器为 service load balancer 创建了一个 Pod，这个 Pod 是[Service](https://kubernetes.io/docs/concepts/services-networking/service/)类型的 Kubernetes 对象。

对于每个 `service load balancer`，都会创建一个[DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)。 DaemonSet 在每个节点上创建一个前缀为svc的 Pod。

Service LB 控制器会监听其他 Kubernetes Services。当它找到一个 Service 后，它会在所有节点上使用 DaemonSet 为该服务创建一个代理 Pod。这个 Pod 成为其他 Service 的代理，例如，来自节点上 8000 端口的请求可以被路由到端口 8888 上的工作负载。[ 代理 Pod 主要的作用就是把流量转发到对应的Workload中 ]

- 如果 Service LB 运行在有外部 IP 的节点上，则使用外部 IP。
- 如果创建多个 Services，则为每个 Service 创建一个单独的 DaemonSet。
- 只要使用不同的端口，就可以在同一节点上运行多个 Services。

如果您尝试创建一个在 80 端口上监听的 Service LB，Service LB 将尝试在集群中找到 80 端口的空闲主机。如果该端口没有可用的主机，LB 将保持 Pending 状态。

### 1.3.2 Service LB 用法

在 K3s 中创建一个 [LoadBalancer 类型的 Service](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer)。

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name:  nginx-deployment
  namespace: default
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name:  nginx
        image: kingsd/nginx:install-tools
        ports:
        - containerPort: 80
          name: nginx-port
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: default
spec:
  selector:
    app: nginx
  type: LoadBalancer
  ports:
  - name: nginx
    protocol: TCP
    port: 8888
    targetPort: nginx-port
$ kubectl create -f nginx-loadbalancer.yaml
deployment.apps/nginx-deployment created
service/nginx created

# svclb是分别部署到两个节点中
$ kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE    IP           NODE   NOMINATED NODE   READINESS GATES
svclb-nginx-wffxt                   1/1     Running   0          119s   10.42.0.17   k3s1   <none>           <none>
nginx-deployment-6b769f4497-x82sp   1/1     Running   0          119s   10.42.0.16   k3s1   <none>           <none>
svclb-nginx-s5jqw                   1/1     Running   0          119s   10.42.1.4    k3s2   <none>           <none
[......]

$ kubectl get svc nginx
NAME    TYPE           CLUSTER-IP     EXTERNAL-IP           PORT(S)          AGE
nginx   LoadBalancer   10.43.92.152   10.0.0.51,10.0.0.52   8888:30285/TCP   40s

$ curl 10.0.0.51:8888 # curl 10.0.0.52:8888 效果是一样的
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
# 查看相应的防火墙流量转发策略
$ iptables -vnL -t nat | grep "8888"
    0     0 CNI-HOSTPORT-SETMARK  tcp  --  *      *       10.42.0.0/24         0.0.0.0/0            tcp dpt:8888
    0     0 CNI-HOSTPORT-SETMARK  tcp  --  *      *       127.0.0.1            0.0.0.0/0            tcp dpt:8888
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8888 to:10.42.0.17:8888
    0     0 CNI-DN-0313432e9c8f2f5b81fd0  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* dnat name: "cbr0" id: "3402365beb39d9f523f6cde07f2915c127643effe8473ec0c82cb077ea3d0eb0" */ multiport dports 8888
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.42.0.0/16         10.43.16.201         /* default/nginx:nginx cluster IP */ tcp dpt:8888
    0     0 KUBE-SVC-GFECHBCWDTHJWART  tcp  --  *      *       0.0.0.0/0            10.43.16.201         /* default/nginx:nginx cluster IP */ tcp dpt:8888
    0     0 KUBE-FW-GFECHBCWDTHJWART  tcp  --  *      *       0.0.0.0/0            10.0.0.51            /* default/nginx:nginx loadbalancer IP */ tcp dpt:8888
    0     0 KUBE-FW-GFECHBCWDTHJWART  tcp  --  *      *       0.0.0.0/0            10.0.0.52            /* default/nginx:nginx loadbalancer IP */ tcp dpt:8888
$ kubectl get pod -o wide -A | grep 10.42.0.17
default       svclb-nginx-wffxt                         1/1     Running     0          3m3s   10.42.0.17   k3s1   <none>           <none>

$ kubectl exec -it svclb-nginx-wffxt -- /bin/sh -c 'iptables -L -t nat | grep "8888"'
DNAT       tcp  -- !nginx.default.svc.cluster.local  anywhere             tcp dpt:8888 to:10.43.16.201:8888

$ kubectl get svc | grep "10.43.16.201"
nginx        LoadBalancer   10.43.16.201   10.0.0.51,10.0.0.52   8888:30467/TCP   3m52s

# 查看Service的详细信息显示
$ kubectl describe svc nginx
Name:                     nginx
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=nginx
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.43.16.201
IPs:                      10.43.16.201
LoadBalancer Ingress:     10.0.0.51, 10.0.0.52
Port:                     nginx  8888/TCP
TargetPort:               nginx-port/TCP
NodePort:                 nginx  30467/TCP
Endpoints:                10.42.0.16:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
# 查看 Endpoints 详细信息显示
$ kubectl describe endpoints nginx
Name:         nginx
Namespace:    default
Labels:       <none>
Annotations:  endpoints.kubernetes.io/last-change-trigger-time: 2023-03-27T00:07:11+08:00
Subsets:
  Addresses:          10.42.0.16
  NotReadyAddresses:  <none>
  Ports:
    Name   Port  Protocol
    ----   ----  --------
    nginx  80    TCP

Events:  <none>

$ kubectl get pod -o wide | grep "10.42.0.16"
nginx-deployment-6b769f4497-x82sp   1/1     Running   0          4m46s   10.42.0.16   k3s1   <none>           <none
```

### 1.3.3 🎓从节点中排除 Service LB

要排除节点使用 Service LB，请将以下标签添加到不应排除的节点上：

```shell
svccontroller.k3s.cattle.io/enablelb
```

如果使用标签，则 `service load balancer` 仅在标记的节点上运行。

```shell
# 查看Node的标签
$ kubectl get nodes --show-labels
NAME   STATUS   ROLES                  AGE     VERSION         LABELS
k3s2   Ready    <none>                 3m23s   v1.21.14+k3s1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=k3s,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k3s2,kubernetes.io/os=linux,node.kubernetes.io/instance-type=k3s
k3s1   Ready    control-plane,master   164m    v1.21.14+k3s1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=k3s,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k3s1,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=true,node-role.kubernetes.io/master=true,node.kubernetes.io/instance-type=k3s
# 设置Node的标签
$ kubectl label node k3s2 svccontroller.k3s.cattle.io/enablelb=true
node/k3s2 labeled

# 再次查看Node的标签
$ kubectl get nodes --show-labels
NAME   STATUS   ROLES                  AGE     VERSION         LABELS
k3s1   Ready    control-plane,master   164m    v1.21.14+k3s1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=k3s,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k3s1,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=true,node-role.kubernetes.io/master=true,node.kubernetes.io/instance-type=k3s
k3s2   Ready    <none>                 3m56s   v1.21.14+k3s1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=k3s,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k3s2,kubernetes.io/os=linux,node.kubernetes.io/instance-type=k3s,svccontroller.k3s.cattle.io/enablelb=true

# 会发现已经没有k3s2的svclb的Pod
$ kubectl get pods -o wide | grep "svclb-nginx"
svclb-nginx-fznbq                   1/1     Running   0          35s    10.42.1.7    k3s2   <none>           <none>
```

### 1.3.4 🎓禁用 Service LB

要禁用嵌入式 LB，请使用`--disable servicelb`选项运行 k3s server。

如果您希望运行其他 LB，例如 MetalLB [ 四层负载均衡器 ] ，这是必需的。

## 1.4 没有主机名的节点

一些云提供商（如 Linode）会以 "`localhost` "作为主机名创建机器，而其他提供商可能根本没有设置主机名。这可能会导致域名解析的问题。你可以用`--node-name`标志或`K3S_NODE_NAME`环境变量来运行 K3s，这样就会传递节点名称来解决这个问题。

- `指定主机名`：

```bash
# K3S_URL : 指定K3s Master URL
# K3S_TOKEN : 指定K3s Master Token[cat /var/lib/rancher/k3s/server/token]
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  K3S_URL=https://10.0.0.51:6443 \ 
  K3S_TOKEN=rancher sh -s - --node-name k3s2
# K3S_URL : 指定K3s Master URL
# K3S_TOKEN : 指定K3s Master Token[cat /var/lib/rancher/k3s/server/token]
# 使用环境变量的方式指定主机名最后还是会以参数的形式进行修改
$ curl -sfL https://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | \
  K3S_NODE_NAME="k3s2" \
  INSTALL_K3S_MIRROR=cn \
  K3S_URL=https://10.0.0.51:6443 \
  K3S_TOKEN=rancher sh -
$ kubectl get nodes
NAME   STATUS   ROLES                  AGE   VERSION
k3s1   Ready    control-plane,master   72m   v1.25.6+k3s1
k3s2   Ready    <none>                 28s   v1.25.6+k3s1
```