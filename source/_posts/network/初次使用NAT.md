---
title: 初次使用NAT
index_img: /images/bg/k8s.webp
banner_img: /images/bg/5.jpg
tags:
  - network
categories:
  - kubernetes
date: 2023-08-09 15:50:12
excerpt: 使用NAT
sticky: 1
hide: true
---


腾讯云服务器A上安装一个docker，运行一个Nginx服务，配置NAT可以通过公网IP直接访问到该容器内部网络


docker run --net=none -itd --name busybox-test busybox
docker exec busybox-test ip a


要使用`ip`命令配置NAT规则，使得能够通过公网IP直接访问到云服务器A上运行的Nginx容器内部网络，需要进行以下步骤：

1. 确保云服务器A已经安装了Docker，并运行了Nginx容器。

2. 配置SNAT规则：

``` bash
# iptables查看NAT规则
$ iptables -t nat -L

# iptables -t nat -A POSTROUTING -s <Nginx容器的IP地址> ! -o docker0 -j SNAT --to-source <云服务器A的公网IP>
$ iptables -t nat -A POSTROUTING -s 10.1.1.8 ! -o br-test -j SNAT --to-source 10.0.8.4


# iptables -t nat -A PREROUTING -d <云服务器A的公网IP> -p tcp --dport <对外暴露的端口号> -j DNAT --to-destination <Nginx容器的IP地址>:80
$ iptables -t nat -A PREROUTING -d 10.0.8.4 -p tcp --dport 8888 -j DNAT --to-destination 10.1.1.8:80

$ iptables -t nat -A OUTPUT -d 10.1.1.8 -p tcp --dport 80 -j DNAT --to-destination 10.0.8.4:8888


$ iptables -t nat -nvL


$ ip netns exec test-nat tcpdump -nei eth0


$ iptables -t nat -A PREROUTING -d 10.1.1.8 -p tcp --dport 80 -j DNAT --to-destination 10.0.8.4:8888

```

3. 验证可以通讯

``` bash
$ docker exec busybox-test nc -l -p 80

# 本机访问
$ echo "hello" | nc 10.1.1.8 80

$ echo "hello" | nc 10.0.8.4 8888

# 公网IP访问
$ echo "hello" | nc 43.138.41.53 8888
```
这些命令将会：

- 添加一条默认路由到名为"nginx"的路由表中，指向云服务器A的网关IP；
- 添加一条规则，将从Nginx容器IP地址发送的流量路由到"nginx"路由表；
- 添加一条默认路由到"nginx"路由表，指向云服务器A的网关IP；
- 添加一条iptables的SNAT规则，将Nginx容器的IP地址的流量源地址转换成云服务器A的公网IP；
- 添加一条iptables的DNAT规则，将外部访问云服务器A公网IP指定端口的流量目标地址转换成Nginx容器的IP地址。

1. 保存配置：将以上配置保存到文件，以使其在重启后自动生效。

``` bash
sudo sh -c "iptables-save > /etc/iptables/rules.v4"
```

这条命令将iptables规则保存到`/etc/iptables/rules.v4`文件中。

完成以上步骤后，就使用`ip`命令成功配置了NAT规则。现在可以通过云服务器A的公网IP和指定的端口号，直接访问到Nginx容器内部网络中运行的Nginx服务了。



ip nat inside source表示转换IP包的源地址，当数据包从内部发往外部；或者转换IP包的目的地址，当数据包从外部传输到内部。
Ip nat outside source表示转换IP包的源地址，当数据包从外部传输到内部；或者转换IP包的目的地址，当数据包从内部发往外部。


静态NAT 一个私有IP固定映射一个公有IP地址，提供内网服务器的对外访问服务
动态NAT 私有IP映射地址池中的公有IP，映射关系是动态的，临时的
NAPT 私有IP地址和端口号与同一个公有地址加端口进行映射


[网络地址转换--静态NAT](https://blog.csdn.net/weixin_42442713/article/details/80909546)


### 如何删除iptables NAT规则

要删除iptables nat规则，需要执行以下步骤：

1. 首先，使用以下命令查看当前的iptables nat规则：

```shell
iptables -t nat -L
```

1. 找到要删除的规则，然后使用以下命令删除指定规则。假设要删除编号为1的规则，使用以下命令：
```shell
iptables -t nat -D PREROUTING 1

iptables -t nat -D OUTPUT 1
```
其中，`PREROUTING`是iptables nat表的一个链，`1`是要删除规则的编号。

1. 重复上述步骤，直到删除所有需要的规则。

请注意，删除规则后，iptables nat表的规则编号会重新调整。因此，如果你想要删除多个规则，最好使用`-L`命令查看规则编号的变化，并相应调整删除的规则编号。

另外，上述命令需要使用root权限运行，或者在sudoers文件中有相应的权限设置。