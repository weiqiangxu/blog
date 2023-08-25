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

##### 1.什么是TUN设备

在计算机网络中，TUN 与 TAP 是操作系统内核中的虚拟网络设备。

- tun是网络层的虚拟网络设备，可以收发第三层数据报文包，如IP封包，因此常用于一些点对点IP隧道等。

- tap是链路层的虚拟网络设备，等同于一个以太网设备，它可以收发第二层数据报文包，如以太网数据帧。Tap最常见的用途就是做为虚拟机的网卡，因为它和普通的物理网卡更加相近，也经常用作普通机器的虚拟网卡。

用户空间的程序可以通过 TUN/TAP 设备发送数据。常见于基于TUN/TAP设备实现的VPN。比如VPN软件在用户空间创建一个TUN/TAP设备，并将其配置为将网络流量导入到VPN隧道中。然后，VPN软件可以通过TUN/TAP设备读取和写入数据，将它们加密并通过隧道发送到VPN服务器。在服务器端，VPN软件将收到的数据解密并通过TUN/TAP设备发送到网络接口，从而实现了VPN连接。

##### 2.特点

TUN：三层设备、IP数据包、实现三层的ip隧道
TAP：二层设备、MAC地址、通常接入到虚拟交换机(bridge)上作为局域网的一个节点

##### 3.隧道

Linux 原生支持多种多种层隧道，大部分底层实现原理都是基于 tun 设备。我们可以通过命令 ip tunnel help 查看 IP 隧道的相关操作。

Linux 原生一共支持 5 种 IP 隧道（常见的隧道协议）。

ipip：即 IPv4 in IPv4，在 IPv4 报文的基础上再封装一个 IPv4 报文。
gre：即通用路由封装（Generic Routing Encapsulation），定义了在任意一种网络层协议上封装其他任意一种网络层协议的机制，IPv4 和 IPv6 都适用。
sit：和 ipip 类似，不同的是 sit 是用 IPv4 报文封装 IPv6 报文，即 IPv6 over IPv4。
isatap：即站内自动隧道寻址协议（Intra-Site Automatic Tunnel Addressing Protocol），和 sit 类似，也是用于 IPv6 的隧道封装。
vti：即虚拟隧道接口（Virtual Tunnel Interface），是 cisco 提出的一种 IPsec 隧道技术。

##### 4.用途

1. VPN连接：可以将tun设备配置为VPN客户端或服务器，并通过该设备在不同网络之间建立安全的隧道连接，实现远程访问或局域网间互通。
2. 隧道连接：可以将tun设备配置为网络隧道的一部分，用于将数据从一个网络传输到另一个网络，通常用于连接不同物理网络的互联，如通过互联网连接不同地区的局域网。
3. 虚拟化网络：可以使用tun设备实现虚拟化网络，通过创建多个tun设备和对应的网络命名空间，可以将不同容器或虚拟机之间隔离的网络连接起来。
4. 流量监控和过滤：可以使用tun设备来捕获传入和传出的网络流量，并进行流量监控或过滤，例如实现防火墙功能等。


[tun-tap工作层图](/images/Tun-tap-osilayers-diagram.png)

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
ip netns exec container1 ip link set lo up
ip netns exec container2 ip addr add 10.1.1.7/24 dev veth4
ip netns exec container2 ip link set veth4 up
ip netns exec container2 ip route add default via 10.1.1.1
ip netns exec container2 ip link set lo up
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
ip netns exec container1 ping 10.0.8.4 -c 3

# ip netns exec container1 ping <同交换机switch\bridge网段容器ip>
ip netns exec container1 ping 10.1.1.7 -c 3
```

### 三、配置TUN的IP隧道

``` bash
# ip tuntap add dev tun1 mode tun
ip netns exec container1 ip tuntap add dev tun1 mode tun

# ip addr add <IP地址>/<子网掩码> dev tun0
ip netns exec container1 ip addr add 172.16.0.6/24 dev tun1

# ip link set dev tun1 up
ip netns exec container1 ip link set dev tun1 up

ip netns exec container1 ping 172.16.0.6 -c 3

# 没有数据流经tun1
ip netns exec container1 tcpdump -nei tun1
```

``` bash
# ip tuntap add dev tun2 mode tun
ip netns exec container2 ip tuntap add dev tun2 mode tun

# set up tun2
ip netns exec container2 ip link set dev tun2 up

# ip addr add <IP地址>/<子网掩码> dev tun2
ip netns exec container2 ip addr add 172.16.0.8/24 dev tun2
```

``` bash
# ping <B的隧道IP地址>
# 验证container1和container2之间通讯
ip netns exec container1 ping 172.16.0.8
```

> 通过TUN的IP隧道，在物理网络上构建一条加密隧道。


### 四、程序监听TUN设备数据

``` golang
package main

import (
	"log"

	"github.com/songgao/packets/ethernet"
	"github.com/songgao/water"
)

func main() {
	config := water.Config{
		DeviceType: water.TUN,
	}
	config.Name = "tun1"

	ifCe, err := water.New(config)
	if err != nil {
		log.Fatalf("err=%s", err)
	}
	var frame ethernet.Frame

	for {
		frame.Resize(1500)
		n, err := ifCe.Read([]byte(frame))
		if err != nil {
			log.Fatalf("read catch=%s", err)
		}
		frame = frame[:n]
		log.Printf("Dst: %s\n", frame.Destination())
		log.Printf("Src: %s\n", frame.Source())
		log.Printf("Ethertype: % x\n", frame.Ethertype())
		log.Printf("Payload: %s\n", string(frame.Payload()))
	}
}
```

### 五、tun设备数据转tap经vrouter三层转发

### 相关疑问

- 客户端使用openvpn访问web服务流程

[openvpn访问过程](https://opengers.github.io/openstack/openstack-base-virtual-network-devices-tuntap-veth/)

- ipv4转发打开持久

``` bash
vim /etc/sysctl.conf

net.ipv4.ip_forward=1
```

- 为什么监听container1的网卡veth2时候，container1 ping无输出而container2 ping有输出

``` bash
# veth2 的 ip 10.1.1.5
# listen veth2 
ip netns exec container1 tcpdump -nei veth2

# 无数据包
ip netns exec container1 ping 10.1.1.5

# 有数据包
ip netns exec container2 ping 10.1.1.5

# lo接口本机ping又有
ip netns exec container1 tcpdump -nei lo
ip netns exec container1 ping 127.0.0.1
```

tcpdump 只能捕获进入它所在网络命名空间的接口的数据包，而无法捕获离开它所在网络命名空间的接口的数据包。

### 相关文档

- [https://cizixs.com/2017/09/28/linux-vxlan/](https://cizixs.com/2017/09/28/linux-vxlan/)
- [什么是 IP 隧道，Linux 怎么实现隧道通信？](https://cloud.tencent.com/developer/article/1432489)
- [Tun/Tap接口使用指导](https://cloud.tencent.com/developer/article/1680749)
- [云计算底层技术-虚拟网络设备(tun/tap,veth)](https://opengers.github.io/openstack/openstack-base-virtual-network-devices-tuntap-veth/)
- [TUN接口有什么用？](https://www.baeldung.com/linux/tun-interface-purpose)
- [Linux虚拟网络基础——tun](https://blog.csdn.net/weixin_39094034/article/details/103810351)
- [Tun/Tap 接口教程](https://backreference.org/2010/03/26/tuntap-interface-tutorial/index.html)