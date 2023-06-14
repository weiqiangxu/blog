---
title: ovs的qos什么
index_img: /images/bg/network.png
banner_img: /images/bg/computer.jpeg
tags:
  - openvswitch
  - docker
categories:
  - openvswitch
date: 2023-06-05 18:40:12
excerpt: ovs的qos的小实验
sticky: 1
---


### 1. 实验环境准备

[ovs实验基础环境](https://weiqiangxu.github.io/2023/06/14/cni/ovs实验基础环境/)

> OVS 的 bond 绑定多个物理网卡成一个逻辑网卡，可以提高网络的可靠性和带宽的实现

``` bash
# 开启网卡
ip link add tab1 type veth peer name peer-tab1
ip link add tab2 type veth peer name peer-tab2
ip link set tab1 up
ip link set peer-tab1 up
ip link set tab2 up
ip link set peer-tab2 up

# 创建一个名为"bond1"的链路聚合组(bond)
# 并将两个网络命名空间中的虚拟网卡"veth1-ns2"和"veth1-ns3"与该链路聚合组绑定，从而实现高可用性和负载均衡
ovs-vsctl add-bond ovs-br1 bond1 tab1 tab2

# 创建一个名为"bond2"的链路聚合组(bond)
# 并将两个网络命名空间中的虚拟网卡"veth1-ns4"和"veth1-ns5"与该链路聚合组绑定，从而实现高可用性和负载均衡
ovs-vsctl add-bond ovs-br2 bond2 peer-tab1 peer-tab2

# 在ovs-br1交换机中，将名称为"bond1"的链路聚合组(bond)的LACP协议设置为主动模式(active)
# LACP为链路聚合控制协议(Link Aggregation Control Protocol)的缩写
# 它用于在链路聚合组成员之间进行协商和控制，以确保链路聚合组的高可用性和负载均衡。
# 在主动模式下，链路聚合组成员会发送LACP协议数据单元，与其他成员进行协商和控制，以建立和维护链路聚合组。这样可以更好地保证链路聚合组的可靠性和性能。
ovs-vsctl set port bond1 lacp=active
ovs-vsctl set port bond2 lacp=active

# Open vSwitch（OVS）命令行工具（ovs-appctl）的一个命令
# 用于显示指定绑定（bond）的状态信息。绑定是指将多个物理网络接口（NIC）组合成一个逻辑接口，以提高带宽和可用性
# bond/show 命令将显示绑定的名称、状态、绑定的物理接口等详细信息。
ovs-appctl bond/show

# 显示Open vSwitch上可用的链路聚合控制协议(Link Aggregation Control Protocol，LACP)的状态和统计信息
# 其中，"ovs-appctl"是Open vSwitch的一个管理工具，用于管理Open vSwitch的各个组件。"lacp/show"是该工具的一个子命令，用于显示LACP相关的信息
# 列出Open vSwitch上所有的链路聚合组及其成员端口、LACP协议状态、LACP协议计数器等信息，从而帮助管理员监控和诊断链路聚合组的状态和性能。
# 例如，可以使用该命令检查链路聚合组是否正常工作、成员端口是否正确配置和活动、链路聚合协议的运行状况等。
ovs-appctl lacp/show


# 默认bond_mode是active-backup就会出现上面的情况
# bond_mode设为balance-tcp\balance-slb
# ovs-vsctl工具设置一个名为bond1的端口的bond模式为balance-slb
# 意思是将bond1端口与其它物理端口进行绑定，以实现网络负载均衡的目的
# 当使用这种模式时，网络流量会被平均地分配到不同的物理端口上，从而提高网络传输效率和可靠性
ovs-vsctl set Port bond1 bond_mode=balance-slb


# 这句话是指使用ovs-vsctl工具设置一个名为bond2的端口的bond模式为balance-tcp
# 意思是将bond2端口与其它物理端口进行绑定，以实现网络负载均衡的目的
# 当使用这种模式时，网络流量会被基于TCP会话的负载均衡算法分配到不同的物理端口上，从而提高网络传输效率和可靠性
# 与balance-slb相比，balance-tcp更加适用于长时间运行的TCP连接，可以保证连接的稳定性和可靠性。
ovs-vsctl set Port bond2 bond_mode=balance-tcp
```

![bonding拓扑图示例](/images/bond.png)

### 2. 验证qos对流量限流

``` bash
# tap1-b2的网络接口上添加Traffic Control（TC）队列规则
# 使用HTB（Hierarchical Token Bucket）算法，将1号句柄作为根规则，并将12号句柄设置为默认规则
tc qdisc add dev peer-tab1 root handle 1: htb default 12


# 这句命令表示在peer-tab1设备上创建一个父类1,子类1:1，并且定义它们的带宽控制规则
# 其中htb表示采用层次令牌桶算法进行带宽控制
# rate表示限制该类最大可用的数据传输速率为100kbps，ceil表示该类最大可用带宽上限也为100kbps。
tc class add dev peer-tab1 parent 1: classid 1:1 htb rate 100kbps ceil 100kbps

# 在peer-tab1网卡上添加一个分类器，指定该分类器的父类别为1:1，类别ID为1:10，
# 采用htb算法进行流量控制，设定该类别的最大带宽为100kbps，
# 但是该类别的最小保障带宽为30kbps，即使存在其他高优先级流量，该类别的带宽也不低于30kbps。
tc class add dev peer-tab1 parent 1:1 classid 1:10 htb rate 30kbps ceil 100kbps

tc class add dev peer-tab1 parent 1:1 classid 1:11 htb rate 10kbps ceil 100kbps
tc class add dev peer-tab1 parent 1:1 classid 1:12 htb rate 60kbps ceil 100kbps
tc class show dev peer-tab1


# 这条命令是为名为 peer-tab1 的网络接口添加队列规则
# 在父队列1:10下创建一个子队列20:
# 该子队列的队列调度算法为pfifo（先进先出），队列长度的最大限制为5
# 这意味着如果队列中的分组数达到5，新的分组将被丢弃。
tc qdisc add dev peer-tab1 parent 1:10 handle 20: pfifo limit 5
tc qdisc add dev peer-tab1 parent 1:11 handle 30: pfifo limit 5
tc qdisc add dev peer-tab1 parent 1:12 handle 40: sfq perturb 10


# 为名为peer-tab1的网络接口添加过滤器规则
# 它将过滤器规则附加到1:0的父队列上，并设置过滤器规则的优先级为1
# 它使用u32匹配模式，匹配源IP地址为192.168.101.3和目标端口号为9999且屏蔽掉后4位（即对65520取模的余数）
# 并将匹配结果定向到子队列1:10上，这意味着所有匹配到该规则的流量都将进入子队列1:10中进行处理。
tc filter add dev peer-tab1 protocol ip parent 1:0 prio 1 u32 match ip src 192.168.101.3 match ip dport 9999 0xfff0 flowid 1:10


# 在peer-tab1网卡上添加一个过滤器，指定该过滤器的协议为ip，父类别为1:0，
# 优先级为1，u32代表使用32位的过滤条件，匹配源IP地址为192.168.101.4的流量，并将它们流向1:11类别。
tc filter add dev peer-tab1 protocol ip parent 1:0 prio 1 u32 match ip src 192.168.101.4 flowid 1:11

# 观察1:11流量
watch tc -s class ls dev peer-tab1
```

``` bash
# 容器删除
docker rm ns4 -f 

# 重建
docker run -it -d --net ovs-net --ip 192.168.101.4 -p 9999:9999 --name ns4 alpine-ovs 
ip link set down veth1f36d25 && ip link set veth1f36d25 name veth1-ns4 && ip link set  veth1-ns4 up
ovs-vsctl del-port ovs-br2 veth1-ns4
ovs-vsctl add-port ovs-br2 veth1-ns4

# 压测速度
# 在名为ns3的容器中运行命令
# 命令是iperf客户端向IP地址为192.168.101.4，端口号为9999的服务器发送1个连接，并在20秒内进行带宽测试。
docker exec -it ns3 sh -c 'iperf -c 192.168.101.4 -p 9999 1 -t 20'
```



### Q&A

1. tc qdisc add dev peer-tab1 parent 1:10 handle 20: pfifo limit 5 这句话什么意思

这句话是 Linux 中的网络命令，用于添加一个队列规则。具体解释如下：
- tc：网络命令的关键字，表示 Traffic Control。
- qdisc：Queueing Discipline 的缩写，表示队列规则。
- add：表示添加一个新的队列规则。
- dev peer-tab1：指定规则所适用的网络接口名称，这里是 peer-tab1。
- parent 1:10：指定父队列的 ID，这里是 1:10，表示这个规则是属于 ID 为 1:10 的父队列下的子队列。
- handle 20:：指定队列的标识符，这里是 20。
- pfifo：选择了一种队列算法，即先进先出队列（Priority-based First-In, First-Out）。
- limit 5：指定队列最大容量为 5 个数据包。如果队列已满，新来的数据包将被丢弃。

2. tc定义队列规则有什么用

  TC（Traffic Control）定义队列规则的作用是为了控制网络流量，防止网络拥塞，保证网络服务的质量和稳定性。通过设置队列规则，可以限制网络中各种流量（如数据包大小、传输速率等）的数量和优先级，以确保不同类型的流量能够有序地传输，并且不会对其他流量产生影响。这对于网络管理员来说是非常重要的管理工具，可以帮助管理员更好地控制网络流量，保证网络的正常运行。


3. iperf如何使用

