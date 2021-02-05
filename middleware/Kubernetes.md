# Kubernetes

- [Kubernetes](#kubernetes)
  - [服务质量等级](#服务质量等级)
  - [kube-proxy](#kube-proxy)
  - [架构](#架构)
    - [核心组件](#核心组件)
  - [Pod生命周期](#pod生命周期)
    - [Pod状态](#pod状态)
    - [容器状态](#容器状态)
    - [容器重启策略](#容器重启策略)
    - [容器探针](#容器探针)
      - [readinessProbe](#readinessprobe)
      - [livenessProbe](#livenessprobe)
      - [startupProbe](#startupprobe)
    - [Pod 的终止](#pod-的终止)
    - [Init 容器](#init-容器)

## 服务质量等级

- Guaranteed

- Burstable

- BestEffort

## kube-proxy

- [iptables概念](https://www.zsythink.net/archives/1199)

- [理解kubernetes环境的iptables](https://www.cnblogs.com/charlieroro/p/9588019.html)

- [kubernetes的Kube-proxy的转发规则分析](https://www.lijiaocn.com/%E9%A1%B9%E7%9B%AE/2017/03/27/Kubernetes-kube-proxy.html)

- [kube-proxy的ipvs模式解读](https://segmentfault.com/a/1190000016333317)

<!-- ## ingress -->

- [为什么我们重新写了一个 Kubernetes Ingress Controller？](http://www.dockone.io/article/9601)

- [Kubernetes Ingress 控制器的技术选型技巧](https://www.jiqizhixin.com/articles/2020-03-24-8)

## 架构

![](https://kubernetes.io/images/blog/2018-06-05-11-ways-not-to-get-hacked/kubernetes-control-plane.png)

### 核心组件

- etcd保存了整个集群的状态。
- apiserver提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制。
- controller manager负责维护集群的状态，比如故障检测、自动扩展、滚动更新等。
- scheduler负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上。
- kubelet负责维护容器的生命周期，同时也负责Volume（CVI）和网络（CNI）的管理。
- Container runtime负责镜像管理以及Pod和容器的真正运行（CRI）。
- kube-proxy负责为Service提供cluster内部的服务发现和负载均衡。

## [Pod生命周期](https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-lifecycle/#termination-of-pods)

### Pod状态

- Pending（悬决） Pod 已被 Kubernetes 系统接受，但有一个或者多个容器尚未创建亦未运行。此阶段包括等待 Pod 被调度的时间和通过网络下载镜像的时间。
- Running（运行中） Pod 已经绑定到了某个节点，Pod 中所有的容器都已被创建。至少有一个容器仍在运行，或者正处于启动或重启状态。
- Succeeded（成功） Pod 中的所有容器都已成功终止，并且不会再重启。
- Failed（失败） Pod 中的所有容器都已终止，并且至少有一个容器是因为失败终止。也就是说，容器以非 0 状态退出或者被系统终止。
- Unknown（未知） 因为某些原因无法取得 Pod 的状态。这种情况通常是因为与 Pod 所在主机通信失败。

### 容器状态

- Waiting （等待）
- Running（运行中）
- Terminated（已终止） 

### 容器重启策略

- Always
- OnFailure 
- Never

### 容器探针

#### readinessProbe

> 确定容器是否已经就绪可以接受流量

#### livenessProbe

> 确定何时重启容器

#### startupProbe

> 指示容器中的应用是否已经启动(针对启动慢的容器)

### Pod 的终止

- preStop钩子 
- 体面终止限期是默认值

### Init 容器