# k8s 浅入门（Apple Silicon）

## 前情提要
对互联网云服务等行业有过一定了解的朋友，肯定都知道掌握k8s容器化技术，是踏入云原生大门的敲门砖。k8s 全称 Kubernetes，是谷歌公司基于自家多年的生产经验，并结合社区中最先进的理念和实践，打造出来的一款开源系统，主要用于自动化容器化应用的部署、扩展和管理。笔者以手头上的一台 Macbook Air 为准，对 k8s 进行简单的入门学习。
## 准备工作
1. 首先选择适合你自己的一款代码编辑器，因为 k8s 的各种资源都是通过 YAML 文件定义的，而 YAML 对缩进极其严格，如果用纯终端工具编辑会比较麻烦。我自己使用的是 VS Code + YAML 扩展插件。
2. 其次还需要部署好本地的 Docker 环境，建议放弃沉重的 Docker Desktop，在 Mac 上使用更轻量、底层网络优化更好的 **OrbStack** 作为容器引擎。
3. 最后安装 k8s 的两个核心命令行工具：`kubectl` 和 `kind` 。
    可以在 Mac 系统的终端里使用 Homebrew 一键安装：
    ```Bash
    brew install kubectl kind
    ```

## 正式开始
### [一、拉起集群与初次部署](1.md)

### [二、核心对象与网络进阶 (Core Objects & Networking)](2.md)

### [三、SRE 可观测性实践 (Observability Ecosystem)](3.md)

### [四、自动化与 GitOps 交付](4.md)

## 结语
- 完成到这里，我们已经走完了 Kubernetes **核心调度与管控** 的基础闭环，触碰到了“原生 K8s ”的边界，
- 在真实的生产环境中， K8s 只是一个底座，大厂会在这个底座上搭建一整套 **云原生生态 (Cloud Native Ecosystem)** 。
- 在 Apple Silicon 架构上，还可以有更进阶的研究方向：
    - 真正的本地负载均衡 (Load Balancer)：部署 **MetalLB** 接管局域网集群
    - 生产级可观测平台 (Monitoring & Alerting)：部署 **Prometheus + Grafana** 自动实时监控
    - 微服务的高级流量治理 (Service Mesh)：部署 **Istio** 服务网格
- 然而本地 Apple Silicon 架构想要完全模拟生产环境，还是有明显的上限的；例如涉及到真实的物理高可用 (HA)、云厂商的深度绑定以及海量并发压测等等。
