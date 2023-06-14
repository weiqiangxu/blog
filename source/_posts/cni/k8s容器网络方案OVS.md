---
title: k8s容器网络解决方案OVS
index_img: /images/bg/k8s.webp
banner_img: /images/bg/5.jpg
tags:
  - openvswitch
categories:
  - kubernetes
date: 2023-06-02 16:56:12
excerpt: 在docker的环境下，如何用ovs创建的网桥接管容器之间的流量，并且验证ovs的一些功能如FlowTable\Tag\trunks等
sticky: 1
---

### 1. 安装ovs

[openvswitch安装](https://weiqiangxu.github.io/2023/06/02/cni/openvswitch%E5%AE%89%E8%A3%85/)

### 2. 安装docker

[docker离线安装](https://weiqiangxu.github.io/2023/04/18/%E8%AF%AD%E9%9B%80k8s%E5%9F%BA%E7%A1%80%E5%85%A5%E9%97%A8/docker%E7%A6%BB%E7%BA%BF%E5%AE%89%E8%A3%85/)

### 3. 制作镜像方便测试

``` dockerfile
# alpine-ovs
FROM alpine:3.16.0

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apk/repositories && apk add vim \
tcpdump && apk add iperf && apk add iproute2
```

### 4. ovs建桥接管容器之间流量

``` bash
# 创建网桥并且指定子网掩码(网段)
$ docker network  create --subnet=192.168.101.0/24 ovs-net
```

``` bash
# 启动3个容器并且指定IP
$ docker run -it -d --net ovs-net --ip 192.168.101.1 --name ns1 alpine-ovs sh
$ docker run -it -d --net ovs-net --ip 192.168.101.2 --name ns2 alpine-ovs sh
$ docker run -it -d --net ovs-net --ip 192.168.101.3 --name ns3 alpine-ovs sh
```

此时3个容器可以互相通讯

``` bash
# 改名容器的网卡对端(插在宿主机上的网卡名称)
# 使用ifconfig可以查看到

ip link set down ${ns1_default_if} && ip link set ${ns1_default_if} name veth1-ns1 && ip link set  veth1-ns1 up
ip link set down ${ns2_default_if} && ip link set ${ns2_default_if} name veth1-ns2 && ip link set  veth1-ns2 up
ip link set down ${ns3_default_if} && ip link set ${ns3_default_if} name veth1-ns3 && ip link set  veth1-ns3 up
```

``` bash
# 查看网桥
# 会发现网桥后面有3个网络插口
brctl show

# 现在我们把这3个网卡从docker创建的网桥拔出来
brctl delif ${bridge_name} veth1-ns1 veth1-ns2 veth1-ns3
```

``` bash
# 使用 ovs 创建的网桥
ovs-vsctl add-br ovs-br1

# 将3个容器的对端网卡插入到 ovs网桥
ovs-vsctl add-port ovs-br1 veth1-ns1
ovs-vsctl add-port ovs-br1 veth1-ns2
ovs-vsctl add-port ovs-br1 veth1-ns3
```

``` bash
# 显示连接到ovs桥的所有端口以及任何流表规则
# 可查看网桥上的所有端口
ovs-ofctl show ovs-br1
```

此时，ovs已经接管了容器之间的网络

``` bash
# 以 OpenFlow 协议显示交换机 ovs-br1 上的流表规则
# 结果中 n_bytes 是指流表规则中匹配到的数据包的字节数，它来自于流表规则的统计信息
# n_bytes 不断增大表示又流量经过 ovs-br1
ovs-ofctl dump-flows ovs-br1
```

### 4. ovs的flow table丢弃icmp数据包

``` bash
# 来体验一下流表规则的作用

# OVS网桥ovs-br1上添加流规则
# 当接收到的数据包是ICMP协议类型，并且其输入端口为1时，将该数据包丢弃（即不处理，不向其他端口转发）
# 该流规则的优先级为3
# 通过 ovs-ofctl show ovs-br1 可以查看ovs桥上面的1\2\3端口是什么
ovs-ofctl add-flow ovs-br1 icmp,priority=3,in_port=1,actions=drop

# 添加以后ping不通ns1了
$ docker exec -it ns2  ping ${ns1_ip}

# ns2 ping ns3 的时候输入端口是3那么不会丢弃报文
$ docker exec -it ns2  ping ${ns3_ip}
PING 192.168.101.3 (192.168.101.3): 56 data bytes
64 bytes from 192.168.101.3: seq=0 ttl=64 time=0.075 ms
64 bytes from 192.168.101.3: seq=1 ttl=64 time=0.062 ms


# 删除刚刚添加的流表规则 <keyword匹配规则即可>
# ovs-ofctl del-flows ovs-br1 <keyword>
ovs-ofctl del-flows ovs-br1 "in_port=veth1-ns1"

# 直接删除流表也将导致无法数据通讯
# 这步骤可以不做验证
ovs-ofctl del-flows ovs-br1
```

### 5. ovs的Port mirroring端口镜像复制端口输入和输出流量至其他端口

``` bash
# 在上面的环境之中将ovs网桥ovs-br1上面的3个插口全部拔出
# ovs-vsctl del-port <bridge> <port>
ovs-vsctl del-port ovs-br1 veth1-ns1
ovs-vsctl del-port ovs-br1 veth1-ns2
ovs-vsctl del-port ovs-br1 veth1-ns3

# 重新将网卡插上并且打上tag
ovs-vsctl add-port ovs-br1 veth1-ns1 -- set Port veth1-ns1 tag=110
ovs-vsctl add-port ovs-br1 veth1-ns2 -- set Port veth1-ns2 tag=110
ovs-vsctl add-port ovs-br1 veth1-ns3 -- set Port veth1-ns3 tag=110

ovs-vsctl list mirror
ovs-vsctl show


# 语法将 vlan 下的所有流量镜像输出到 output-port
ovs-vsctl set Bridge <bridge> mirrors=@m -- --id=@<mirror> create Mirror name=<mirror_name> select-all=true select-vlan=<vlan_id> output-port=<mirror_port>

# 创建一个名为“m1”的镜像，并将其名称分配给变量“@m”。 
# 然后，它创建两个"id"，分别为“@veth1-ns1”和“@veth1-ns4”，这两个"id"指定了要镜像的端口
# 接下来，命令使用这些"id"来设置镜像规则。
# 它选择数据包的目标端口，并将其复制到输出端口“@veth1-ns3”
# 同时选择“veth1-ns1”端口作为源地址。这意味着所有发送到“veth1-ns1”的数据包都将被复制到“veth1-ns3”。
ovs-vsctl -- set bridge br1 mirrors=@m \
          -- --id=@veth1-ns1 get Port veth1-ns1 \
          -- --id=@veth1-ns2 get Port veth1-ns2 \
          -- --id=@m create Mirror name=m1 select-dst-port=@veth1-ns1 select-src-port=@veth1-ns1 output-port=@veth1-ns3


# select-dst-port用于匹配数据包的目的地端口，选择应该将数据包发送到哪个端口。
# select-src-port用于匹配数据包的源端口，选择哪个端口接收数据包

# 监听网卡 output-port 也就是 veth1-ns3
tcpdump -nei veth1-ns3

# ns1 ping ns2 , 查看流量是否 mirro 到 veth1-ns3
docker exec -it ${ns1_id} ping ${ns2_ip}
```

``` bash
# 设置 Open vSwitch 网桥（bridge） ovs-br1 的镜像规则
# 创建了一个名为 m1 的镜像规则，将 veth1-ns2 端口的流量作为镜像目标（select-dst-port），并将同样的流量作为镜像源（select-src-port）；
# 为了将镜像流量和普通流量区分开来，还将镜像流量打上 VLAN 110 的标签（output-vlan）。
ovs-vsctl -- set bridge ovs-br1 mirrors=@m2 \
          -- --id=@veth1-ns1 get Port veth1-ns1 \
          -- --id=@m2 create Mirror name=m1 select-dst-port=@veth1-ns2 select-src-port=@veth1-ns2 output-vlan=111


# 设置 Open vSwitch 网桥（bridge） br2 的镜像规则
# 创建了一个名为 m3 的镜像规则，将选取 VLAN ID 为 110 的流量作为镜像源（select-vlan）
# 并将同样的流量打上 VLAN 110 的标签作为镜像流量（output-vlan）
# 将这个镜像规则赋值给 mirrors 属性，即将该规则应用于 ovs-br1 上。
ovs-vsctl -- set bridge ovs-br1 mirrors=@m \
          -- --id=@m create Mirror name=m3 select-vlan=110 output-vlan=110


# 1. 指定洪泛的VLAN ID，使bridge只会向这些VLAN上的所有端口广播，而不会向其他VLAN广播。
# 2. 在多个VLAN之间提供隔离，从而限制数据包传播的范围。
# 3. 支持多租户场景下的VLAN隔离，不同租户可以独立使用自己的VLAN。
# 4. 防止恶意攻击者利用洪泛攻击来干扰网络的正常运行。
#VLAN隔离和洪泛控制
ovs-vsctl set bridge br1 flood-vlans=110

# 监听网卡 veth1-ns4
tcpdump -nei veth1-ns4

# ns2 ping ns3
docker exec -it ${ns2_id} ping <ns3_ip>
```

``` bash
# 环境清理
ovs-vsctl clear bridge ovs-br1 mirrors
ovs-vsctl clear bridge ovs-br1 mirrors
```

### 6. tag设置相同bridge下的不同vlan无法通信

``` bash
# 在上面的环境之中将ovs网桥ovs-br1上面的3个插口全部拔出
# ovs-vsctl del-port <bridge> <port>
ovs-vsctl del-port ovs-br1 veth1-ns1
ovs-vsctl del-port ovs-br1 veth1-ns2
ovs-vsctl del-port ovs-br1 veth1-ns3

# 重新将网卡插上并且打上tag
ovs-vsctl add-port ovs-br1 veth1-ns1 -- set Port veth1-ns1 tag=110
ovs-vsctl add-port ovs-br1 veth1-ns2 -- set Port veth1-ns2 tag=110
ovs-vsctl add-port ovs-br1 veth1-ns3 -- set Port veth1-ns3 tag=111

# 从 ns1 ping ns3
docker exec -it ${ns1_id} ping <ns3_ip>
```

### 6. trunks && flood-vlans

``` bash

# 这句话的意思是将名称为"veth1-ns1"的端口设置为trunk端口
# 允许通过VLAN ID为110和111的多个VLAN
# 它的作用是在Open vSwitch中配置端口，使其能够处理来自多个VLAN的数据包
ovs-vsctl set Port veth1-ns1 trunks=110,111

# 不同tag(vlan)之间网络是否通
docker exec -it ${ns1_id} ping <ns3_ip>


# 在Open vSwitch中设置名称为"ovs-br1"的网桥支持洪泛（flood）VLAN 110和111
# 作用是在VLAN架构中使用洪泛技术向所有主机广播数据包
# 由于洪泛可能会导致网络中的广播风暴，因此只有在局域网或者小规模网络中才应该使用
# 洪泛技术可以使网络中所有的设备都能够接收到广播数据包，从而实现网络通信
ovs-vsctl set bridge ovs-br1 flood-vlans=110,111


# 不同tag(vlan)之间网络是否通
docker exec -it ${ns1_id} ping <ns3_ip>
```


### Q&A

- 开启洪泛和 MAC 学习什么关系

   开启洪泛（Flood）是一种网络攻击手段，可以让网络中的所有数据包都被广播，从而导致网络拥塞甚至瘫痪。
   禁止MAC学习是一种网络安全配置策略，可以限制网络中的MAC地址学习，防止网络中的恶意设备伪装成其他合法设备进行攻击。

### 相关文章

[基于openvswitch实现的openshit-sdn](https://zhuanlan.zhihu.com/p/37852626)
[kubernetes 网络组件简介（Flannel & Open vSwitch & Calico）](https://blog.csdn.net/kjh2007abc/article/details/86751730)
