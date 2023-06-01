---
hide: true
---

### 一、CNI


### 二、网络相关术语

1. ingrees egress
2. attach
3. tc
4. ifup && ifdown
5. command `ip route` 
6. command `route -n`


### Q&A

- route -n 指令如何解析结果

``` bash
[root@i-7B581709 ~]# route -n
# 查看到默认网关数据转发通过 br-ext 网桥转发流量
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.16.207.254   0.0.0.0         UG    0      0        0 br-ext  （当前系统默认的网关）
10.16.200.0     0.0.0.0         255.255.248.0   U     0      0        0 br-ext   （目标网络）
169.254.0.0     0.0.0.0         255.255.0.0     U     0      0        0 br-ext    （目标网络）
192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 virbr0    （本地网络）


这是一份路由表，显示了当前系统的网络路由信息。具体解析如下：

- 第一列 Destination： 目标网络或主机的IP地址或地址范围，0.0.0.0 表示默认的路由，即默认的网关。
- 第二列 Gateway：下一跳路由器的IP地址，用于将数据包转发到目标网络或主机。
- 第三列 Genmask：子网掩码，用于指定目标网络或主机的范围（即该地址范围所包含的IP地址）。
- 第四列 Flags：路由标志，用于标识路由的属性和状态。
- 第五列 Metric：路由的距离度量值，用于指定优先级，通常是跃点数或延迟等。
- 第六列 Ref：路由的引用计数，表示有多少个套接字正在使用这个路由。
- 第七列 Use：路由的使用计数，表示已经经过该路由的数据包数量。
- 第八列 Iface：该路由所对应的网络接口。

根据这份路由表，可以知道
当前系统默认的网关是 10.16.207.254，
目标网络为 10.16.200.0/21 和 169.254.0.0/16
以及本地网络 192.168.122.0/24 所对应的网络接口为 virbr0。

- Flags的UG和U的意思：
  UG 表示为 Up/下一跳为网关
  U 表示Up/可用
```

- ip route 结果解析

``` bash
[root@i-7B581709 ~]# ip route
# 默认路由，目标IP转发到固定网关 - 主要为了查看这个
default via 10.16.207.254 dev br-ext 
10.16.200.0/21 dev br-ext proto kernel scope link src 10.16.203.160 
169.254.0.0/16 dev br-ext scope link 
192.168.122.0/24 dev virbr0 proto kernel scope link src 192.168.122.1 linkdown 

# 路由表信息
- "default via 10.16.207.254 dev br-ext" 表示默认路由，即如果目标IP地址没有匹配到任何已知的路由，则会将数据包发送到10.16.207.254网关设备，通过br-ext接口发送出去。
- "10.16.200.0/21 dev br-ext proto kernel scope link src 10.16.203.160" 表示直接连通网络，即10.16.200.0/21这个子网段可以直接通过br-ext接口访问，本机IP地址是10.16.203.160。
- "169.254.0.0/16 dev br-ext scope link" 表示自动配置IPv4地址（APIPA地址）的网络，即当本机无法获取到有效IP地址时，会自动分配一个169.254.x.x的IP地址，并可以在br-ext接口上进行通信。
- "192.168.122.0/24 dev virbr0 proto kernel scope link src 192.168.122.1 linkdown" 表示在虚拟网络设备virbr0上的子网，本机通过这个设备连接到虚拟机。但是因为该设备处于linkdown（未连接）状态，无法进行通信。
```

- 查看防火墙服务是否开启

``` bash
$ systemctl -l | grep firewalk
$ systemctl stop firewalld
```

- 查看某一个ip是否可以访问

``` bash
$ ping 192.168.1.23
```

- 查看某一个ip的某一个端口是否可以访问

``` bash
$ telnet 127.0.0.1 8881
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
```

- 端口是否有服务在监听

``` bash
$ netstat -tunlp | grep 8881
tcp 0 0 0.0.0.0:8881 0.0.0.0:* LISTEN 21748/ovsdb-server 

-t：显示TCP协议的连接情况
-u：显示UDP协议的连接情况
-n：显示IP地址和端口号，而不是域名和服务名
-l：显示监听状态的连接情况
-p：显示与连接相关的进程PID和进程名

$ netstat -nl | grep 8881
tcp 0 0 0.0.0.0:8881 0.0.0.0:* LISTEN
```

### 相关资料

[深入K8S组网架构和Flannel原理](https://juejin.cn/post/6884881812145995790)
[https://github.com/cilium](https://github.com/cilium)
[https://cilium.io/](https://cilium.io/)
[https://docs.cilium.io/en/stable/](https://docs.cilium.io/en/stable/)
[https://hub.docker.com/r/cilium/cilium](https://hub.docker.com/r/cilium/cilium)
[使用 Cilium 作为网络插件部署 K8s + KubeSphere](https://zhuanlan.zhihu.com/p/471220158)
[在K8s集群上部署Istio的三种方式](https://zhuanlan.zhihu.com/p/135298580)
[Linux ip 命令](https://www.runoob.com/linux/linux-comm-ip.html)