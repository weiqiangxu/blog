---
title: 什么是CNI
index_img: /images/bg/k8s.webp
banner_img: /images/bg/5.jpg
tags:
  - kubernetes
categories:
  - kubernetes
date: 2023-06-05 18:40:12
excerpt: 理解什么是CNI，k8s的网络模型是什么，实现的插件有哪些，相关的OVS又是什么
sticky: 1
---

### 一、简短描述

1. CNCF项目;
2. 标准化的网络接口规范;
3. CNI规范定义了"容器运行时"如何与"网络插件"进行通信，并且规定了插件必须实现的功能和接口;
3. 第三方插件: 
     [calico](https://github.com/projectcalico/calico)
     [cilium](https://github.com/cilium/cilium)

### 二、k8s和CNI是什么关系

- Kubernetes遵循CNI（Container Networking Interface）规范；
- 使用CNI插件来实现容器间通信和与外部网络通信，定义和管理容器的网络配置；
- Kubernetes提供了一些默认的CNI网络插件，例如Flannel和Calico等；
- CNI网络抽象层


### 三、k8s如何集成flannel

- 官方手册关于网络插件安装的指引在[使用kubeadm引导集群](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/);

``` bash
# 在控制平面节点或具有 kubeconfig 凭据的节点上安装 Pod 网络附加组件
# 每个集群只能安装一个 Pod 网络
$ kubectl apply -f <add-on.yaml>
```

### 四、CNI标准和CNI提供的二进制程序(什么关系)

> CNI插件是符合CNI规范的二进制程序

### 五、只有 [CNI/plugins](https://github.com/containernetworking/plugins) 可以安装好k8s的网络吗

   不是的，除了 https://github.com/containernetworking/plugins 还有很多其他的网络插件可以用于Kubernetes，比如：Calico、Flannel、Weave Net、Cilium等等。可以根据自己的需求选择适合的网络插件来部署Kubernetes集群。

### Q&A

##### 1.k8s使用ovs通信的时候，当pod与pod之间通信，数据流向怎么样的

> OVS充当着虚拟交换机的角色，Kubernetes网络模型插件协调多个节点上不同的OVS实现容器之间通信。

- 1.应用程序在一个pod中生成数据包，数据包被发送到pod的网络接口;
- 2.Kubernetes网络模型插件（例如：Flannel等）将数据包封装在一个Overlay协议中;
- 3.CNI插件比如flannel将数据包发送到虚拟交换机（OVS）;
- 4.OVS使用VXLAN或者Geneve等封装技术，将数据包发送到另一个pod的OVS;
- 5.另一个pod的OVS解开数据包，将数据包发送到目标pod的网络接口;
- 6.应用程序在目标pod中接收到数据包。

##### 2.单价的kubernetes如何运行非控制平面的pod

默认情况下，出于安全原因，你的集群不会在控制平面节点上调度 Pod。 参考官方手册[控制平面节点隔离](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)，如果你希望能够在控制平面节点上调度 Pod，例如单机 Kubernetes 集群：

``` bash
$ kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

##### 3.kubelet管理CNI插件吗

- Kubernetes 1.24 之前，CNI 插件也可以由 kubelet 使用命令行参数 cni-bin-dir 和 network-plugin 管理;
- Kubernetes 1.24 移除了这些命令行参数， CNI 的管理不再是 kubelet 的工作;

##### 4.如何安装和管理CNI插件

理解[网络模型](https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-networking-model) - 如何实现网络模型 - [安装扩展](https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/addons/#networking-and-network-policy) - [使用helm or kubectl 安装 flannel](https://github.com/flannel-io/flannel#deploying-flannel-manually)

##### 5.CNI插件和flannel什么关系

- cni插件和flannel都是用于容器网络的技术。
- cni插件用于管理容器网络，flannel则是一种网络解决方案，可以提供容器间的通信。
- cni插件可以与flannel结合使用，以管理flannel提供的网络。在使用kubernetes等容器编排工具时，通常都会使用cni插件和flannel进行容器网络的管理。

##### 6.kube-proxy和flannel是什么关系

   kube-proxy和flannel可以说是两个不同的组件，但它们是紧密相关的。
   kube-proxy作为Kubernetes中的一种网络代理，负责管理集群内部的网络连接。
   flannel则是一种网络解决方案，用于管理容器之间的通信。
   在一个Kubernetes集群中，flannel会被作为网络解决方案被部署，并使用kube-proxy来实现网络代理的功能。
   flannel和cni也有关系。cni是Container Network Interface的缩写，是用于实现容器网络的通用API规范。
   而flannel则是cni规范的一个实现。也就是说，在一个Kubernetes集群中，flannel可以被作为cni规范的一个实现来使用，以实现容器网络的功能。

##### 7.kube-proxy是每个节点都有吗，控制节点还是工作节点，独立进程还是pod内还是二进制程序而已

   kube-proxy是每个工作节点都有的，它作为一个独立的进程运行在每个节点上。
   它的主要作用是实现Kubernetes服务发现和负载均衡机制，为Service对象提供集群内服务的访问入口。
   kube-proxy会在节点上创建iptables规则或操作网络设备，以实现Service IP的转发和负载均衡。

##### 8.flannel是CNI的实现吗

   flannel是一种CNI（Container Networking Interface）的实现，用于容器之间的网络通信。
   它使用覆盖网络的技术来实现容器之间的通信，支持多种网络模型，如host-gw、vxlan、ipip等，可以在不同节点之间建立虚拟网络。
   flannel的主要作用是为容器提供独立的IP地址，并实现跨节点的通信，让容器能够像在一个本地网络中一样相互通信。

##### 9.CNI仓库之中有CNI规范库\CNI插件是什么关系，CNI插件(bridge\firewall\ptb等)和flannel是什么关系

- CNI规范是一种容器网络接口规范，定义了容器运行时与网络插件之间的接口规范;
- CNI插件(轻量级网络插件)是遵循CNI规范定义的一种网络插件，用于在容器中创建和管理网络接口;
- Flannel是一个常用的CNI插件之一，用于创建虚拟网络，并为容器提供IP地址，管理集群网络和跨节点通信;
- 在使用Flannel时，可以选择使用CNI插件作为网络接口与容器运行时交互，以实现跨主机容器的网络通信；

> CNI插件和Flannel是互补的关系

##### 10.kube-proxy原理是什么

[k8s/参考/组件工具/kube-proxy](https://kubernetes.io/zh-cn/docs/reference/command-line-tools-reference/kube-proxy/)

- 网络代理在每个节点上运行；
- 执行 TCP、UDP 和 SCTP 流转发；
- 转发是基于iptables的（用iptables规则来实现Kubernetes Service的负载均衡和端口转发功能），每个Service创建一条iptables规则；

### 相关资料

[官方手册 https://github.com/containernetworking/cni](https://github.com/containernetworking/cni)
[官方手册 CNI/plugins && https://github.com/containernetworking/plugins](https://github.com/containernetworking/plugins)
[k8s/概念/扩展/网络插件](https://kubernetes.io/zh-cn/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
[k8s/概念/集群管理/集群网络系统/网络模型 && 如何实现网络模型](https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-networking-model)
[k8s/概念/集群管理/安装扩展](https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/addons/#networking-and-network-policy)
