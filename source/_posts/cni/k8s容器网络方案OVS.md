---
title: k8s容器网络解决方案OVS
index_img: /images/bg/k8s.webp
banner_img: /images/bg/5.jpg
tags:
  - openvswitch
categories:
  - kubernetes
date: 2023-06-02 16:56:12
excerpt: 介绍OVS是什么，来一个quick start体验一下
sticky: 1
---

### 1. 安装ovs

[openvswitch安装](https://weiqiangxu.github.io/2023/06/02/cni/openvswitch%E5%AE%89%E8%A3%85/)

### 2. 安装docker

[docker离线安装](https://weiqiangxu.github.io/2023/04/18/%E8%AF%AD%E9%9B%80k8s%E5%9F%BA%E7%A1%80%E5%85%A5%E9%97%A8/docker%E7%A6%BB%E7%BA%BF%E5%AE%89%E8%A3%85/)

### 3. ovs如何接管docker流量
### 4. ovs的flow table丢弃icmp数据包
### 5. tag设置相同bridge下的不同vlan无法通信
### 6. trunks 将多个虚拟局域网（VLANs）互相隔离的通信链路连接在一起
### 7. flood-vlans 泛洪控制不通vlan之间流量

### 相关文章

[基于openvswitch实现的openshit-sdn](https://zhuanlan.zhihu.com/p/37852626)
[kubernetes 网络组件简介（Flannel & Open vSwitch & Calico）](https://blog.csdn.net/kjh2007abc/article/details/86751730)
