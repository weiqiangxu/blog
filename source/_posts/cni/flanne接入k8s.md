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


### 3.flannel组件

- Cni0网桥
- 每创建一个pod都会创建一对 veth pair，一端是pod中的eth0，另一端在Cni0网桥中的端口 （网卡）
- Flannel.1（overlay网络设备，vxlan报文的处理（封包和解包），node之间pod流量从overlay设备以`隧道的形式`发送到对端）

``` bash
# ifconfig命令可以输出网络接口信息，包括网卡和网桥
$ ifconfig
cni0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.244.0.1  netmask 255.255.255.0  broadcast 10.244.0.255
        inet6 fe80::d40e:d7ff:fe08:ea95  prefixlen 64  scopeid 0x20<link>
        ether d6:0e:d7:08:ea:95  txqueuelen 1000  (Ethernet)
        RX packets 6024367  bytes 486510605 (463.9 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 5923922  bytes 559849199 (533.9 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.244.0.0  netmask 255.255.255.255  broadcast 0.0.0.0
        inet6 fe80::c82c:7dff:fec9:888  prefixlen 64  scopeid 0x20<link>
        ether ca:2c:7d:c9:08:88  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 8 overruns 0  carrier 0  collisions 0
```

### 4.flannel网络通信原理实验

1. 同节点的pod之间通信

   在容器启动前，会为容器创建一个虚拟Ethernet接口对，这个接口对类似于管道的两端，其中一端在主机命名空间中，另外一端在容器命名空间中，并命名为eth0。
   在主机命名空间的接口会绑定到网桥。网桥的地址段会取IP赋值给容器的eth0接口。

> 利用nodeName特性将Pod调度到相同的node节点上（亲和性）


``` bash
$ cat >p1.yaml<<EOF
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: p1
  namespace: default
spec:
  nodeName: k8s-node1
  containers:
  - image: busybox
    name: c1
    command: 
    - "ping"
    - "baidu.com"
EOF

$ cat >p2.yaml<<EOF
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: busybox
  name: p2
  namespace: default
spec:
  nodeName: k8s-node1
  containers:
  - image: busybox
    name: c2
    command: 
    - "ping"
    - "baidu.com"
EOF
```
``` bash
$ kubectl apply -f p1.yaml -f p2.yaml
$ kubectl get pod p{1,2}
$ kubectl get pod p{1,2} -o wide
# p1 ping p2
$ kubectl exec -it p1 -- ping 10.244.1.230


# 源容器向目标容器发送数据，数据首先发送给 cni0 网桥

$ kubectl exec -it p1 -c c1 -- ip route
$ kubectl exec -it p2 -c c2 -- ip route

# cni0 网桥接受到数据后，将其转交给 flannel.1 虚拟网卡处理

# flannel.1 接受到数据后，对数据进行封装，并发给宿主机的 ens33，再通过ens33出外网，或者访问不同node节点上的pod
```

![想同节点的不同pod之间的通信-基于flannel](/images/flannel-same-pod.png)

2. 不同节点的pod之间通信

3. pod与service之间的通信

### 相关资料

- [Kubernetes（k8s）CNI（flannel）网络模型原理](https://mp.weixin.qq.com/s/18bMpQjXFodfegWH3xNeKQ)
