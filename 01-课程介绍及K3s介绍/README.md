Video Reference：[01-课程介绍及K3s介绍](https://www.bilibili.com/video/BV1yR4y1g7Ud/?spm_id_from=333.1007.top_right_bar_window_history.content.click&vd_source=9560c118fae1db9638f05a6ba2527085)

GitHub README.md：[01-课程介绍及K3s介绍](https://github.com/kingsd041/k3s-tutorial/blob/main/01-课程介绍及K3s介绍/README.md)

## 1 课程介绍

本教程是 K3s 的系列教程，会按照[K3s 官网](https://docs.rancher.cn/k3s/)的目录顺序来针对 K3s 的每个功能做讲解和操作，以便大家更深入了解 K3s。

当 K3s 新版本发布时，会针对每个版本做详细的介绍。

## 2 K3s 介绍

[K3s - 轻量级 Kubernetes:](https://docs.rancher.cn/docs/k3s/_index)

- CNCF认证的Kubernetes发行版
- 支持 X86_64, ARM64, ARMv7平台
- 单一进程包含 Kubernetes master，kubelet，和 containerd

**生命周期:** K3s 会同时支持 3 个 K8s 版本，支持的生命周期与 K8s 相同，参考: [Kubernetes 版本及版本偏差支持策略](https://kubernetes.io/zh/docs/setup/release/version-skew-policy/)

**v1.20.4+k3s1**: `v1.20.4` 为 K8s 版本，`k3s1` 为补丁版本

**更新周期**: 当 K8s 更新新版本后，一般情况下，K3s 在一周内同步更新。

**latest/stable/testing**: https://update.k3s.io/v1-release/channels

K3s 的启动其实是依赖一个K3s的二进制文件，再结合K3s的安装脚本，可以快速把K3s集群启动起来。

K3s 是一个轻量级的 Kubernetes 发行版，它针对边缘计算、物联网等场景进行了高度优化。K3s 有以下增强功能：

- 打包为单个二进制文件。
- 使用基于 sqlite3 的轻量级存储后端作为默认存储机制。同时支持使用 etcd3、MySQL 和 PostgreSQL 作为存储机制。
- 封装在简单的启动程序中，通过该启动程序处理很多复杂的 TLS 和选项。
- 默认情况下是安全的，对轻量级环境有合理的默认值。[ K3s证书默认是1年的有效期，K3s默认也有证书轮转的机制，再小于90天之内如果重启K3s就会自动轮转K3s证书 ]
- 添加了简单但功能强大的`batteries-included`功能，例如：本地存储提供程序，服务负载均衡器，Helm controller 和 Traefik Ingress controller。
- 所有 Kubernetes control-plane 组件的操作都封装在单个二进制文件和进程中，使 K3s 具有自动化和管理包括证书分发在内的复杂集群操作的能力。
- 最大程度减轻了外部依赖性，K3s 仅需要 kernel 和 cgroup 挂载。 K3s 软件包需要的依赖项包括：

- - Containerd
  - Flannel
  - CoreDNS
  - CNI
  - 主机实用程序（iptables、socat 等）
  - Ingress controller（Traefik）
  - 嵌入式服务负载均衡器（service load balancer）
  - 嵌入式网络策略控制器（network policy controller）

K3s 默认使用的容器运行时环境是 Containerd，可以使用Docker进行替换。

## 3 为什么叫 K3s

我们希望安装的 Kubernetes 在内存占用方面只是一半的大小。Kubernetes 是一个 10 个字母的单词，简写为 K8s。所以，有 Kubernetes 一半大的东西就是一个 5 个字母的单词，简写为 K3s。K3s 没有全称，也没有官方的发音。

## 4 国内资源

中文官网文档：https://docs.rancher.cn/k3s

国内 K3s 资源：http://mirror.cnrancher.com/

## 5 周边项目

- [k3os](https://github.com/rancher/k3os): k3OS是一个完全基于Kubernetes管理的轻量级操作系统，能大大简化在低资源环境中的Kubernetes操作，快速安装，秒级启动。
- [k3d](https://github.com/rancher/k3d): k3d创建容器化的k3s集群。这意味着，您可以使用docker在单台计算机上启动多节点k3s集群。[ Docker in K3s ]
- [autok3s](https://github.com/cnrancher/autok3s): AutoK3s 是用于简化 K3s 集群管理的轻量级工具，您可以使用 AutoK3s 在任何地方运行 K3s，Run K3s Everywhere。
- [octopus](https://github.com/cnrancher/octopus): Octopus是用于Kubernetes和k3s的轻量级云原生设备管理系统，它不需要替换Kubernetes集群的任何基本组件。部署octopus后，集群可以将边缘设备作为自定义k8s资源进行管理。
- [harvester](https://github.com/rancher/harvester): Harvester是基于Kubernetes构建的开源超融合基础架构（HCI）软件。它是vSphere和Nutanix的开源替代方案。

## 6 适用场景

K3s 适用于以下场景：

- 边缘计算-Edge
- 物联网-IoT
- CI
- Development
- ARM
- 嵌入 K8s

由于运行 K3s 所需的资源相对较少，所以 K3s 也适用于开发和测试场景。在这些场景中，如果开发或测试人员需要对某些功能进行验证，或对某些问题进行重现，那么使用 K3s 不仅能够缩短启动集群的时间，还能够减少集群需要消耗的资源。与此同时，Rancher 中国团队推出了一款针对 K3s 的效率提升工具：**AutoK3s**。只需要输入一行命令，即可快速创建 K3s 集群并添加指定数量的 master 节点和 worker 节点。如需详细了解 AutoK3s，请参考[AutoK3s 功能介绍](https://docs.rancher.cn/docs/k3s/autok3s/_index)。

## 7 遇到问题怎么办？

1. k3s github issue: https://github.com/k3s-io/k3s/issues
2. 在此 repo 中创建issue
3. 中文论坛: https://forums.cnrancher.com/
4. Rancher 社区