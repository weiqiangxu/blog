---
title: 集群的clusterIP可以访问吗
index_img: /images/bg/network.png
banner_img: /images/bg/computer.jpeg
tags:
  - kubernetes
categories:
  - kubernetes
date: 2023-09-19 18:40:12
excerpt: k8s网络模型相关
sticky: 1
---

1. 集群的cluser ip 可以直接访问吗
2. network policy
3. describe svc的时候endpoint可以直接访问吗
4. host network模式
5. cluster级别和namespace级别
6. pod的ip和service的cluster ip关系
7. cluster ip 可以访问吗 pod的ip可以访问吗
8. metadata\selector\label分别是干嘛的
9. k8s的cert-manager下的pod是干嘛的（证书的更新、颁发、管理）
10. k8s运行起来需要哪些组件（Mater组件、Node组件）
    ```s
    要在Kubernetes上成功运行应用程序，需要部署以下组件：
    1. Kubernetes Master组件：
      - kube-apiserver：提供Kubernetes API，并处理集群管理的核心功能。
      - kube-controller-manager：负责运行控制器，监控集群状态并处理集群级别的任务。
      - kube-scheduler：负责为Pod选择合适的节点进行调度。

    2. Kubernetes Node组件：
      - kubelet：在每个节点上运行，负责管理和执行Pod的生命周期。
      - kube-proxy：负责实现Kubernetes服务的网络代理和负载均衡功能。
      - 容器运行时：如Docker、containerd等，负责在节点上启动和管理容器。

    3. etcd：分布式键值存储系统，用于保存Kubernetes集群的状态和配置。

    4. Kubernetes网络插件：用于实现Pod之间和Pod与外部网络的通信，常见的插件有Calico、Flannel、Weave等。

    5. 可选组件：
      - kube-dns/coredns：为集群中的服务提供DNS解析。
      - Kubernetes Dashboard：提供Web界面用于管理和监控集群。
      - Ingress Controller：用于处理集群中的入口流量，并将流量路由到相应的服务。

    除了以上核心组件，还可以根据需要添加其他组件和功能，如日志收集器、监控系统等。总之，以上组件是构成一个基本的Kubernetes集群所必需的组件，它们共同协作来实现容器编排和应用程序管理。
    ```
11. k8s的权限管理是怎么样的
12. kube-proxy是干嘛的
13. kube-proxy的源码我应该怎么读，分哪几块理解，kube-proxy的设计是怎么样的
14. k8s初始运行多少个pod
    ```txt
    cert-manager
    kube-system\csi-cephfsplugin
    kube-system\elastic-autoscaler-manager
    kube-systeme\etcd
    kube-system\kube-apiserver
    kube-system\kube-controller-manager
    kube-system\kube-flannel
    kube-system\traefik
    kube-system\web-kubectl
    kube-system\resourcequota-webhook-manage
    ```
15. 运行一个k8s，需要安装在宿主机的软件有哪些，比如cni插件二进制脚本需要安装在宿主机上
    ```bash
    在宿主机上安装和运行Kubernetes（k8s）集群，需要以下软件和工具：

    1. 容器运行时（Container Runtime）：Kubernetes支持多个容器运行时，如Docker、containerd、CRI-O等。你需要在宿主机上安装所选容器运行时，并确保其能与Kubernetes集成。

    2. kubeadm：这是一个用于部署和管理Kubernetes集群的命令行工具，需要在宿主机上进行安装。

    3. kubelet：这是Kubernetes集群中每个节点上的主要组件，负责管理容器的生命周期和运行状态。kubeadm会自动安装和配置kubelet。

    4. kubectl：这是Kubernetes的命令行工具，用于与集群进行交互、管理和监控。你需要在宿主机上安装kubectl。

    5. CNI插件：CNI（Container Network Interface）是Kubernetes网络模型的一部分，它定义了容器网络如何与宿主机和其他容器进行通信。你需要选择一个CNI插件，如Flannel、Calico、Weave等，并将其二进制脚本安装在宿主机上。每个节点上的CNI插件负责为容器提供网络连接。

    此外，如果你使用的是容器运行时Docker，那么你还需要在宿主机上安装Docker Engine。注意，Docker Engine与Docker CLI是两个不同的组件，你只需要安装Docker Engine。
    ```
16. 