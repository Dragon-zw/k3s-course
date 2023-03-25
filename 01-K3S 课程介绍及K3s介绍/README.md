Video Reference：[01-课程介绍及K3s介绍](https://www.bilibili.com/video/BV1yR4y1g7Ud/?spm_id_from=333.1007.top_right_bar_window_history.content.click&vd_source=9560c118fae1db9638f05a6ba2527085)

GitHub README.md：[01-课程介绍及K3s介绍](https://github.com/kingsd041/k3s-tutorial/blob/main/01-课程介绍及K3s介绍/README.md)

## 1 课程介绍

本教程是 K3s 的系列教程，会按照[K3s 官网](https://docs.rancher.cn/k3s/)的目录顺序来针对 K3s 的每个功能做讲解和操作，以便大家更深入了解 K3s。

当 K3s 新版本发布时，会针对每个版本做详细的介绍。

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679725545310-d692f395-f995-4813-be59-3118678d61de.png)

## 2 K3s 介绍和功能

[K3s - 轻量级 Kubernetes:](https://docs.rancher.cn/docs/k3s/_index)

- CNCF 认证的 Kubernetes 发行版
- 支持 `X86_64, ARM64, ARMv7` 平台
- 单一进程包含 Kubernetes master，kubelet，和 containerd

K3s 的启动其实是依赖一个K3s的二进制文件，再结合K3s的安装脚本，可以快速把K3s集群启动起来。

K3s 是一个轻量级的 Kubernetes 发行版，它针对边缘计算、物联网等场景进行了高度优化。K3s 有以下增强功能：

**为你提供 k3s 的产品介绍**

- 打包为单个二进制文件。

- - 把 K8S 相关的组件，比如 kube-api / kube-manager / kube-scheduler 都打包到同一个二进制文件里面，这样的话，只需要启动这个文件就可以快速的启动对应的组件。 

- 使用`基于 sqlite3 的轻量级存储后端作为默认存储机制`。

- - 同时支持使用 etcd3、MySQL 和 PostgreSQL 作为存储机制。

- 封装在简单的启动程序中，通过该启动程序处理很多复杂的 TLS 和选项。
- 默认情况下是安全的，对轻量级环境有合理的默认值。[ K3s证书默认是1年的有效期，K3s默认也有证书轮转的机制，再小于90天之内如果重启K3s就会自动轮转K3s证书 ]

- - 在 K3s 中有一个默认的证书管理机制`(默认一年有效期)`，也有一个可以轮转证书的功能(就是在小于九十天之内重启 K3s 的话，就会自动续一年)。

- 添加了简单但功能强大的`batteries-included`功能，例如：本地存储提供程序，服务负载均衡器，Helm controller 和 Traefik Ingress controller。

- - 就是虽然有些服务本身这个二进制文件并没有提供，但是可以通过内置的服务，将配置文件放到指定的目录下面，就可以在启动的时候一并将该服务启动或替换默认组件。

- 所有 Kubernetes control-plane 组件的操作都封装在单个二进制文件和进程中，

- - 因为封装在二进制文件中，所以启动的时候只有一个进程。好处在于只需要管理这个单一进程就可以了，同时使 K3s 具有自动化和管理包括证书分发在内的复杂集群操作的能力。
  - 可以避免管理员像管理K8s一样需要管理多个组件以及多个进程

- 最大程度减轻了外部依赖性，K3s 仅需要 `kernel` 和 `cgroup` 挂载[ 即稍新一点的 Linux 内核就可以了(需要 kernel 和 cgroup 挂载)。 ]。 K3s 软件包需要的依赖项包括：

- - Containerd [ 可以使用Docker作为容器运行时环境 ]
  - Flannel
  - CoreDNS
  - CNI
  - 主机实用程序（iptables、socat、etc 等）
  - Ingress Controller（Traefik）[ 默认的controller服务(可以使用`apisix/ingress-controller`) ]
  - 嵌入式服务负载均衡器（service load balancer）
  - 嵌入式网络策略控制器（network policy controller）

- **生命周期**

- - 同时支持 3 个 K8s 版本，支持的生命周期与 K8s 相同
  - 可以参考: [Kubernetes 版本及版本偏差支持策略](https://kubernetes.io/zh/docs/setup/release/version-skew-policy/) 进行学习

- **更新周期**

- - 当 K8s 更新新版本后，一般 K3s 在一周内同步更新
  - 可以通过 [这个链接](https://update.k3s.io/v1-release/channels) 获取 `**latest/stable/testing**` 版本
  - 我们默认安装的是 stable 版本，可以运行通过命令进行查看
  - `**latest/stable/testing**`: https://update.k3s.io/v1-release/channels
  - ![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679736562054-b2273d8a-05b5-4025-b6b8-534c6402a2da.png)

- **命名规范**

- - **v1.20.4+k3s1**: v1.20.4  为 K8s 版本，k3s1  为补丁版本

K3s 默认使用的容器运行时环境是 Containerd，可以使用Docker进行替换。

## 3 为什么叫 K3s

我们希望安装的 Kubernetes 在内存占用方面只是一半的大小。Kubernetes 是一个 10 个字母的单词，简写为 K8s。所以，有 Kubernetes 一半大的东西就是一个 5 个字母的单词，简写为 K3s。K3s 没有全称，也没有官方的发音。

之所以叫做 K3S 是因为希望安装的 K8S 在内存占用方面只是一半的大小，而一半大的东西就是一个 5 个字母的单词，简写为 K3S。

## 4 国内资源

中文官网文档：https://docs.rancher.cn/k3s

国内 K3s 资源：https://mirror.rancher.cn/

![img](https://cdn.nlark.com/yuque/0/2023/png/2555283/1679736626651-088a10ac-3eb5-4307-8ca6-d6a75f152ef0.png)

## 5 周边项目

- [k3os](https://github.com/rancher/k3os)

- - k3OS是一个完全基于Kubernetes管理的轻量级操作系统，能大大简化在低资源环境中的Kubernetes操作，快速安装，秒级启动。

- [k3d](https://github.com/rancher/k3d)

- - k3d创建容器化的 K3s 集群。这意味着，您可以使用Docker在单台计算机上启动多节点 K3s 集群。[ Docker in K3s ]
  - 即 K3s 内置到容器中，可以在一台服务器上通过Docker启动多个 K3s 节点

- [autok3s](https://github.com/cnrancher/autok3s)

- - AutoK3s 是用于简化 K3s 集群管理的轻量级工具，您可以使用 AutoK3s 在任何地方运行 K3s。[ Run K3s Everywhere ]
  - 可以到云平台的服务器上使用 AutoK3s

- [octopus](https://github.com/cnrancher/octopus)

- - Octopus是用于 Kubernetes 和 k3s 的轻量级云原生设备管理系统，它不需要替换Kubernetes集群的任何基本组件。
  - 部署octopus后，集群可以将边缘设备作为自定义 k8s 资源进行管理。

- [harvester](https://github.com/rancher/harvester)

- - Harvester 是基于 Kubernetes 构建的开源超融合基础架构（HCI）软件。它是`vSphere`和`Nutanix`的开源替代方案。

## 6 适用场景

K3s 适用于以下场景：

- 边缘计算-Edge
- 物联网-IoT
- CI
- Development
- ARM
- 嵌入 K8s
- 不想深陷 K8s 运维管理的人

由于运行 K3s 所需的资源相对较少，所以 K3s 也适用于开发和测试场景。在这些场景中，如果开发或测试人员需要对某些功能进行验证，或对某些问题进行重现，那么使用 K3s 不仅能够缩短启动集群的时间，还能够减少集群需要消耗的资源。

与此同时，Rancher 中国团队推出了一款针对 K3s 的效率提升工具：**AutoK3s**。只需要输入一行命令，即可快速创建 K3s 集群并添加指定数量的 master 节点和 worker 节点。如需详细了解 AutoK3s，请参考[AutoK3s 功能介绍](https://docs.rancher.cn/docs/k3s/autok3s/_index)。

## 7 遇到问题怎么办？

1. k3s github issue: https://github.com/k3s-io/k3s/issues
2. 在此 repo 中创建 issue
3. 中文论坛: https://forums.cnrancher.com/
4. Rancher 社区

## 8 参考链接

- **[1] 文档教程**

- - [K3s 中文文档 - 国外](http://mirror.cnrancher.com/)
  - [K3s 中文文档 - 国内](https://docs.rancher.cn/k3s)
  - [K3s 国内镜像站 - 加速](https://mirror.rancher.cn/)
  - [K3s 系列教程 - 官方制作](https://github.com/kingsd041/k3s-tutorial)

- **[2] 代码地址**

- - [K3s 仓库地址 - Github](https://github.com/k3s-io)

- **[3] 周边项目**

- - [K3s 周边项目 - k3os](https://github.com/rancher/k3os)

- - - 完全基于 K8S 管理的轻量级操作系统

- - [K3s 周边项目 - autok3s](https://github.com/cnrancher/autok3s)

- - - 用于简化 K3s 集群部署和管理的轻量级工具
    - 即在阿里云和 aws 等云服务器上面部署 k3s

- - [K3s 周边项目 - k3d](https://github.com/rancher/k3d)

- - - 可以在 k3d 创建容器化的 k3s 集群
    - 可以使用容器在单台计算机上启动多节点 k3s 集群

- - [K3s 周边项目 - harvester](https://github.com/rancher/harvester)

- - - 基于 K8S 构建的开源超融合基础架构(HCI)软件
    - 旨在替换 vSphere 和 Nutanix 的开源替代方案

- - [K3s 周边项目 - octopus](https://github.com/cnrancher/octopus)

- - - 主要用于边缘计算相关
    - 用于 K8S 和 k3s 的轻量级云原生设备管理系统
    - 集群可以将边缘设备作为自定义 k8s 资源进行管理