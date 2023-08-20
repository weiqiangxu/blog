---
title: 初次使用NAT
index_img: /images/bg/k8s.webp
banner_img: /images/bg/5.jpg
tags:
  - network
categories:
  - kubernetes
date: 2023-08-20 15:50:12
excerpt: 使用NAT实现数据转发其中包括DNAT和SNAT实验
sticky: 1
hide: true
---

### 一、概念

##### 1.NAT

NAT（网络地址转换）是一种网络技术，一般用于局域网和公网之间IP地址转换，常用iptables实现。

##### 2.DNAT

DNAT（目标网络地址转换）是NAT的一种形式，它将目标IP地址和端口转换为不同的IP地址和端口，通常用于将外部请求转发到内部网络中的特定服务器上。一般通过公网IP进来公网网卡的数据包更改目的ip或端口访问到内部服务。

##### 3.SNAT

SNAT（源网络地址转换）是NAT的另一种形式，它将发送方的IP地址和端口转换为不同的IP地址和端口。主要用于局域网内的多台设备通过同一个公共IP地址访问互联网时，可以使用SNAT将内部设备的源IP地址转换成公共IP地址。这样可以避免互联网上的服务器将响应发送回源IP地址时的冲突。

### 二、配置DNAT规则让外部访问内部网络


1. 购买腾讯云服务器A上安装一个docker，运行一个Nginx服务，配置DNAT可以使用公网IP访问Nginx服务.

``` bash
$ yum install -y docker
$ systemctl start docker
$ docker run -itd --name nginx-test nginx
```

2. 配置NAT使用公网IP与自定义端口可访问Nginx服务

``` bash
# iptables查看NAT规则
$ iptables -t nat -L

# docker容器ip地址
$ docker inspect nginx-test | grep IPAddress

# 配置公网IP与8080端口请求转发到本机80端口
# 10.0.8.4 <公网数据入口网卡IP> 8989 <公网端口号> to-destination <容器IP地址>:<容器端口>
$ iptables -t nat -A PREROUTING -d 10.0.8.4 -p tcp --dport 8989 -j DNAT --to-destination 172.17.0.2:80

# 配置完成后可以通过腾讯云<公网IP>:8989访问到docker服务
# 如何删除iptables规则
$ iptables -t nat -D PREROUTING 1
```

### 三、配置SNAT从容器内部访问外网





### 相关疑问


##### 1.iptables常用命令

``` bash
# iptables查看NAT规则
$ iptables -t nat -L

# 查看iptables规则和其匹配次数
$ iptables -t nat -nvL
```

##### 2.iptables的PREROUTING\POSTROUTING\OUTPUT分别干嘛的

iptables是一个用于Linux系统的防火墙工具，用于配置和管理网络数据包过滤规则。其中的PREROUTING、POSTROUTING和OUTPUT是iptables的三个不同的表，用于不同的数据包处理阶段。

- PREROUTING表: 进入路由系统的数据包。数据包路由之前进行处理，常用目标地址的修改、端口重定向等。

- POSTROUTING表: 离开路由系统的数据包。数据包路由之后进行处理，常用源地址的修改等。常见的使用场景SNAT等。

- OUTPUT表: 本地产生的数据包。它在数据包从本地应用程序发送出去之前进行处理，可以对数据包进行一些操作，例如目标地址的修改、端口重定向等。常见的使用场景包括阻止/允许本地应用程序访问特定的目标地址/端口等。

综上所述，PREROUTING表用于处理进入路由系统的数据包，POSTROUTING表用于处理离开路由系统的数据包，OUTPUT表用于处理本地产生的数据包。

##### 3.什么是静态NAT和动态NAT

一个私有IP固定映射一个公有IP地址，提供内网服务器的对外访问服务是静态。动态NAT是私有IP映射地址池中的公有IP，映射关系是动态的，临时的。

##### 4.如何删除iptables NAT规则

```shell
# 数字1是链的index索引
iptables -t nat -D PREROUTING 1

iptables -t nat -D OUTPUT 1
```

##### 5.本机器curl本机器网卡会经过iptables吗

在本机上使用curl命令访问IP地址为本机网卡IP`10.0.8.4`的服务端口8989，那么这个请求不会经过iptables防火墙。iptables是Linux操作系统中的一个防火墙管理工具，在本机上进行网络请求，请求的目标IP地址是本机的网卡IP地址，那么这个请求是走本机的网络协议栈直接发送和接收的，不会经过iptables的过滤。iptables主要针对通过本机的网络数据流量进行过滤和管理。
