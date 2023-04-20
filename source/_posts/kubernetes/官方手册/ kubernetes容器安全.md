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

``` bash
轻量级的虚拟化技术，可以打包应用程序及其依赖项，使其更易于部署和管理
k8s支持多种容器运行时（Container Runtime），包括Docker、containerd、CRI-O等
```

- 容器运行时Container Runtime是什么意思

``` txt
一种软件，其主要任务是负责在操作系统上启动和管理容器
容器运行时通常通过调用操作系统提供的系统调用 - 来创建和管理容器
一般和容器编排工具（例如Kubernetes）协同工作，实现容器的自动化部署、扩缩容等
常见容器运行时有：Docker容器引擎、rkt、containerd等
```

- Kubernetes中的容器可能会有哪些安全风险

``` txt
1. 容器之间共享主机系统的资源,可能会通过共享文件或进程来获取其他容器中的敏感信息
2. 容器的容量限制不够严格
3. 容器镜像来源不可信
4. 容器网络安全风险,比如需要访问外部网络或者其他容器
5. 容器数据持久化缺少加密
6. Kubernetes容器默认以高权限运行，容器内进程的文件系统和主机文件系统是共享的
```

- 模拟docker的容器A非法访问容器B的资源
- NetworkNamespace是在Linux内核中实现的一种机制，用于隔离网络资源，例如网络接口、路由表和iptables规则等
- kata是什么
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

- linux的命名空间是什么
``` bash
命名空间是Linux内核中的一个概念，它可以将不同的系统资源隔离开来，比如网络、进程空间等。
通过将容器连接到特定的网络命名空间中，可以实现容器与特定网络资源的隔离和互通
```

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
[从零开始入门 K8s | Kata Containers 创始人带你入门安全容器技术](https://zhuanlan.zhihu.com/p/122247284)