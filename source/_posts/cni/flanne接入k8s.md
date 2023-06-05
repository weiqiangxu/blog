---
title: flannel接入k8s
index_img: /images/bg/k8s.webp
banner_img: /images/bg/5.jpg
tags:
  - flannel
categories:
  - kubernetes
date: 2023-04-23 18:40:12
excerpt: kubernetes网络解决方案flannel
sticky: 1
---

### 1.kubernetes如何接入flannale

[k8s/概念/扩展/网络插件](https://kubernetes.io/zh-cn/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
[k8s/概念/扩展/网络插件/containerd安装网络插件](https://github.com/containerd/containerd/blob/main/script/setup/install-cni)
[k8s/概念/扩展/网络插件/CRI-O安装网络插件](https://github.com/cri-o/cri-o/blob/main/contrib/cni/README.md)

> 安装 cni-plugins 网络插件，然后安装 flannel

[k8s/概念/集群管理/安装扩展](https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/addons/#networking-and-network-policy)
[deploying-flannel-with-kubectl](https://github.com/flannel-io/flannel#deploying-flannel-with-kubectl)
[deploying-flannel-with-helm](https://github.com/flannel-io/flannel#deploying-flannel-with-helm)


### 2.flannel通信原理

   Flannel通信原理是虚拟网络技术，使用Linux内核中的TUN/TAP驱动程序创建虚拟网络接口，通过该接口收发数据包。
   Flannel将每个物理机的IP地址映射到虚拟网络中的不同IP地址，使得不同物理机之间可以通过虚拟网络进行通信。
   Flannel还提供了多种底层网络实现，包括UDP、VXLAN、GRE等，以适应不同的网络环境。