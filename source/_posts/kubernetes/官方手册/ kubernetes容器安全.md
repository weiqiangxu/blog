---
title: kubernetes容器安全
index_img: /images/bg/k8s.webp
banner_img: /images/bg/5.jpg
tags:
  - kubernetes
categories:
  - kubernetes
date: 2023-04-18 18:40:12
excerpt: 介绍k8s操作系统底层资源分割逻辑
sticky: 1
---


### 课题

- 容器在k8s是什么
- 什么是容器安全
- 容器逻辑上分割，物理上的资源区隔的设计是什么样的
- kubernetes的安全策略，如容器隔离、网络策略、RBAC设计是什么样的
- iSula是什么
- iSula+StratoVirt安全容器是什么
- 在容器执行top命令看到宿主机进程，为什么
- containerd和docker什么关系，有架构图吗
- runc、cri、运行时是什么
- 容器网络安全是怎么样的
- k8s的设计的架构图
- docker架构图
- Istios是第二代Service Mesh的代表
- Service Mesh服务网格是一种用于解决微服务架构中服务之间通信的问题的技术
- namespace和cgroups标准是什么
- OCI(开放容器标准)是什么涉及哪些内容
- Kubernetes的CRI(Container Runtime Interface)的容器运行时接口是什么意思
- shim的设计:作为适配器将自身容器运行时接口适配到 Kubernetes 的 CRI 接口(dockershim就是Kubernetes对接Docker到CRI接口)
- CGroup是Control Groups限制\记录\隔离进程组所使用的物理资源

### 课题方向

1. 容器哪里不安全了
2. 目前的解决方案是什么样的
3. 解决方案的使用怎样可达到更好的效果
4. 一些常见的兼容性、性能测试覆盖一下


> Containerd 实现了 Kubernetes 容器运行时接口 (CRI)
> BuildKit 是一种开源工具，它从 Dockerfile 获取指令并“构建”Docker 映像

### 架构图

![什么是k8s的CRI-O](/images/什么是k8s的CRI-O.png)
![早期的k8s与docker](/images/早期的k8s与docker.png)
![containerd集成cri-containerd-shim后架构图](/images/containerd集成cri-containerd-shim后架构图.png)
![docker和containerd关系](/images/docker和containerd关系.png)
![docker依赖k8s标准](/images/docker依赖k8s标准.png)
![k8s-v1.20-24分离docker-shim](/images/k8s-v1.20-24分离docker-shim.png)
![k8s-v1.20之前内置docker-shim](/images/k8s-v1.20之前内置docker-shim.png)
![k8s-v1.24之后自行安装cri-dockerd](/images/k8s-v1.24之后自行安装cri-dockerd.png)
![k8s分离docker-shim](/images/k8s分离docker-shim.png)
![k8s与docker分离的初步计划](/images/k8s与docker分离的初步计划.png)
![kubelet和containerd简化调用链过程](/images/kubelet和containerd简化调用链过程.png)
![kubelet与容器运行时](/images/kubelet与容器运行时.png)
![kubelet与cri内部结构](/images/k8s分离docker-shim.png)

### 相关资料

[github.com/containerd](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)
[zhihu/什么是 Service Mesh](https://zhuanlan.zhihu.com/p/61901608)
[PhilCalcado/Pattern: Service Mesh](https://philcalcado.com/2017/08/03/pattern_service_mesh.html)
[官网运行时container-runtimes](https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/)
[csdn剖析容器docker运行时-说的太细致了](https://blog.csdn.net/m0_57776598/article/details/126963904)
[csdn之IaaS/PaaS/SaaS/DaaS的区别-说的太好了](https://blog.csdn.net/yangyijun1990/article/details/108694011)
[知乎/container之runc](https://zhuanlan.zhihu.com/p/279747954)