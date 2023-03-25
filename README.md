# k3s-course
# Rancher K3s 课程学习

K3s 是一个轻量级的 Kubernetes 发行版，它针对边缘计算、物联网等场景进行了高度优化。K3s 有以下增强功能：

- 打包为单个二进制文件。
- 使用基于 sqlite3 的轻量级存储后端作为默认存储机制。同时支持使用 etcd3、MySQL 和 PostgreSQL 作为存储机制。
- 封装在简单的启动程序中，通过该启动程序处理很多复杂的 TLS 和选项。
- 默认情况下是安全的，对轻量级环境有合理的默认值。
- 添加了简单但功能强大的`batteries-included`功能，例如：本地存储提供程序，服务负载均衡器，Helm controller 和 Traefik Ingress controller。
- 所有 Kubernetes control-plane 组件的操作都封装在单个二进制文件和进程中，使 K3s 具有自动化和管理包括证书分发在内的复杂集群操作的能力。
- 最大程度减轻了外部依赖性，K3s 仅需要 kernel 和 cgroup 挂载。 K3s 软件包需要的依赖项包括：
  - containerd
  - Flannel
  - CoreDNS
  - CNI
  - 主机实用程序（iptables、socat 等）
  - Ingress controller（Traefik）
  - 嵌入式服务负载均衡器（service load balancer）
  - 嵌入式网络策略控制器（network policy controller）
  
# 参考官网地址：

  [K3s - 轻量级 Kubernetes | K3s](https://docs.k3s.io/zh/)

  [K3s - 轻量级 Kubernetes | Rancher文档](http://docs.rancher.cn/docs/k3s/_index)

# 学习视频

- [K3s 快闪直播系列 01-课程介绍及 K3s 介绍](https://www.bilibili.com/video/BV1yR4y1g7Ud)
- [K3s 快闪直播系列: 02-K3s 架构及快速入门](https://www.bilibili.com/video/BV14u411m7YT)
- [K3s 快闪直播系列: 03-安装-要求及选项](https://www.bilibili.com/video/BV1Gu411m7vx)
- [K3s 快闪直播系列: 04-安装-K3s Server 和 Agent 配置](https://www.bilibili.com/video/BV1du41127YC)
- [K3s 快闪直播系列: 05-安装-网络选项](https://www.bilibili.com/video/BV1uR4y1M7Nm)
- [K3s 快闪直播系列: 06-08 K3s 高可用安装](https://www.bilibili.com/video/BV1c3411Y7mM)
- [K3s 快闪直播系列: 09-安装-私有镜像仓库配置参考](https://www.bilibili.com/video/BV16m4y1U7LS)
- [K3s 快闪直播系列: 10-安装-离线安装](https://www.bilibili.com/video/BV1Q44y157LM)
- [K3s 快闪直播系列: 11-安装-仪表盘及卸载 K3s](https://www.bilibili.com/video/BV1SL4y1b7g7)
- [K3s 快闪直播系列: 12-升级 k3s 集群](https://www.bilibili.com/video/BV16R4y1g77f)
- [K3s 快闪直播系列: 13-备份和恢复](https://www.bilibili.com/video/BV1Dm4y1U7fw)
- [K3s 快闪直播系列: 14-卷和存储](https://www.bilibili.com/video/BV19r4y1v7UT)
- [K3s 快闪直播系列: 15-网络](https://www.bilibili.com/video/BV1tZ4y1f7Bb)
- [K3s 快闪直播系列: 16-Helm](https://www.bilibili.com/video/BV1wS4y1773w)
- [K3s 快闪直播系列: 17-高级选项和配置](https://www.bilibili.com/video/BV1cS4y177Lt)
