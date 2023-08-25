---
title: 使用tun设备隧道通信
index_img: /images/bg/network.png
banner_img: /images/bg/computer.jpeg
tags:
  - network
categories:
  - network
date: 2023-08-25 18:40:12
excerpt: 部署多个网络命名空间并使用GRE隧道通讯
sticky: 1
hide: false
---

### 一、概念

1. 什么是TUN设备

在计算机网络中，TUN 与 TAP 是操作系统内核中的虚拟网络设备。TAP 等同于一个以太网设备，它操作第二层数据包如以太网数据帧。TUN 模拟了网络层设备，操作第三层数据包比如IP数据封包。操作系统通过 TUN/TAP 设备向绑定该设备的用户空间的程序发送数据，反之，用户空间的程序也可以像操作硬件网络设备那样，通过 TUN/TAP 设备发送数据。在后种情况下，TUN/TAP 设备向操作系统的网络栈投递（或注入）数据包，从而模拟从外部接受数据的过程。

2. 特点

TUN：三层设备、IP数据包、实现三层的ip隧道
TAP：二层设备、MAC地址、通常接入到虚拟交换机(bridge)上作为局域网的一个节点

3. 隧道

Linux 原生支持多种三层隧道，其底层实现原理都是基于 tun 设备。我们可以通过命令 ip tunnel help 查看 IP 隧道的相关操作。

，Linux 原生一共支持 5 种 IP 隧道。

ipip：即 IPv4 in IPv4，在 IPv4 报文的基础上再封装一个 IPv4 报文。
gre：即通用路由封装（Generic Routing Encapsulation），定义了在任意一种网络层协议上封装其他任意一种网络层协议的机制，IPv4 和 IPv6 都适用。
sit：和 ipip 类似，不同的是 sit 是用 IPv4 报文封装 IPv6 报文，即 IPv6 over IPv4。
isatap：即站内自动隧道寻址协议（Intra-Site Automatic Tunnel Addressing Protocol），和 sit 类似，也是用于 IPv6 的隧道封装。
vti：即虚拟隧道接口（Virtual Tunnel Interface），是 cisco 提出的一种 IPsec 隧道技术。



### 二、初始化环境

``` bash
yum install -y bridge-utils
ip netns add container1
ip netns add container2
ip netns list
ip link add veth1 type veth peer name veth2
ip link add veth3 type veth peer name veth4
ip link set veth2 netns container1
ip link set veth4 netns container2
ip netns exec container1 ip addr add 10.1.1.5/24 dev veth2
ip netns exec container1 ip link set veth2 up
ip netns exec container1 ip route add default via 10.1.1.1
ip netns exec container2 ip addr add 10.1.1.7/24 dev veth4
ip netns exec container2 ip link set veth4 up
ip netns exec container2 ip route add default via 10.1.1.1
brctl addbr br-link
brctl addif br-link veth1
brctl addif br-link veth3
ip link set veth1 up
ip link set veth3 up
ip addr add 10.1.1.1/24 dev br-link
ip link set br-link up
ip tuntap help
```

``` bash
# 验证环境已经配置好
# 检查ipv4转发
sysctl net.ipv4.ip_forward

# 打开ipv4转发
sysctl -w net.ipv4.ip_forward=1

# 测试容器之间网络互通
# ip netns exec container1 ping <宿主机eth0>
ip netns exec container1 ping 10.0.8.4

# ip netns exec container1 ping <同交换机switch\bridge网段容器ip>
ip netns exec container1 ping 10.1.1.7
```

### 三、配置TUN的IP隧道

``` bash
# ip tuntap add dev tun0 mode tun
ip netns exec container1 ip tuntap add dev tun0 mode tun

# ip link set dev tun0 up
ip netns exec container1 ip link set dev tun0 up

# ip addr add <IP地址>/<子网掩码> dev tun0
ip netns exec container1 ip addr add 172.16.0.6/24 dev tun0
```

``` bash
# ip tuntap add dev tun1 mode tun
ip netns exec container2 ip tuntap add dev tun1 mode tun

# set up tun1
ip netns exec container2 ip link set dev tun1 up

# ip addr add <IP地址>/<子网掩码> dev tun0
ip netns exec container2 ip addr add 172.16.0.8/24 dev tun1
```

``` bash
# ping <B的隧道IP地址>
# 验证container1和container2之间通讯
ip netns exec container1 ping 172.16.0.8
```

> 通过TUN的IP隧道，在物理网络上构建一条加密隧道。


#### 四、程序监听TUN设备数据

```python
import os
import select
from scapy.all import *

tun = open('/dev/net/tun', 'r+b')
flags = fcntl.fcntl(tun, fcntl.F_GETFL)
fcntl.fcntl(tun, fcntl.F_SETFL, flags | os.O_NONBLOCK)

while True:
   r, w, x = select.select([tun], [], [])
   for fd in r:
       packet = os.read(fd, 2048)
       pkt = Ether(packet)
       pkt.show()
```

### 相关文档

[https://cizixs.com/2017/09/28/linux-vxlan/](https://cizixs.com/2017/09/28/linux-vxlan/)
[什么是 IP 隧道，Linux 怎么实现隧道通信？](https://cloud.tencent.com/developer/article/1432489)