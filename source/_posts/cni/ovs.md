---
title: OVS入门
index_img: /images/bg/k8s.webp
banner_img: /images/bg/5.jpg
tags:
  - openvswitch
categories:
  - kubernetes
date: 2023-06-02 16:56:12
excerpt: 介绍OVS是什么，来一个quick start体验一下
sticky: 1
---

### 一、什么是Open Flow协议

- 网络通信协议，定义了在SDN下网络流量控制机制
- 目标是可编程的、可定制的、可调整的网络解决方案

### 二、启动ovs之后查看相关进程

``` bash
$ ps aux | grep openvswitch

# 查询进程ID为65191的进程打开的文件或网络套接字的命令
$ lsof -p 65191

# 查看进程打开了多少个文件或者网络套接字
$ lsof -p 21764
COMMAND     PID USER   FD      TYPE             DEVICE SIZE/OFF      NODE NAME
ovs-vswit 21764 root  cwd       DIR              253,3     4096       128 /
ovs-vswit 21764 root  rtd       DIR              253,3     4096       128 /
ovs-vswit 21764 root  txt       REG              253,3 16136816   1820044 /usr/sbin/ovs-vswitchd
ovs-vswit 21764 root  mem       REG              253,3  1531880      2546 /usr/lib64/libc-2.28.so
ovs-vswit 21764 root  mem       REG              253,3   724008      2550 /usr/lib64/libm-2.28.so
ovs-vswit 21764 root  mem       REG              253,3    68152      2562 /usr/lib64/librt-2.28.so
ovs-vswit 21764 root  mem       REG              253,3   136544      2558 /usr/lib64/libpthread-2.28.so
ovs-vswit 21764 root  mem       REG              253,3    67640    332375 /usr/lib64/libatomic.so.1.2.0
ovs-vswit 21764 root  mem       REG              253,3   204088      2539 /usr/lib64/ld-2.28.so
ovs-vswit 21764 root    0u      CHR                1,3      0t0      1029 /dev/null
ovs-vswit 21764 root    1u      CHR                1,3      0t0      1029 /dev/null
ovs-vswit 21764 root    2u      CHR                1,3      0t0      1029 /dev/null
ovs-vswit 21764 root    3w      REG              253,3     1941 176160908 /var/log/openvswitch/ovs-vswitchd.log
ovs-vswit 21764 root    4u     unix 0x0000000071d69593      0t0     74075 type=DGRAM
ovs-vswit 21764 root    5uW     REG               0,22        6     75886 /run/openvswitch/ovs-vswitchd.pid
ovs-vswit 21764 root    6w     FIFO               0,12      0t0     74076 pipe
ovs-vswit 21764 root    7w     FIFO               0,12      0t0     75079 pipe
ovs-vswit 21764 root    8r     FIFO               0,12      0t0     75887 pipe
ovs-vswit 21764 root    9w     FIFO               0,12      0t0     75887 pipe
ovs-vswit 21764 root   10u     unix 0x00000000de2593a2      0t0     75888 /var/run/openvswitch/ovs-vswitchd.21764.ctl type=STREAM
ovs-vswit 21764 root   11u  netlink                         0t0     75890 ROUTE
ovs-vswit 21764 root   13u  netlink                         0t0     75893 ROUTE
ovs-vswit 21764 root   14u  netlink                         0t0     75897 ROUTE
ovs-vswit 21764 root   15u  netlink                         0t0     75898 ROUTE
ovs-vswit 21764 root   16u     sock                0,9      0t0     75902 protocol: UDP
ovs-vswit 21764 root   17u  netlink                         0t0     75910 ROUTE
ovs-vswit 21764 root   18u  netlink                         0t0     75911 GENERIC
ovs-vswit 21764 root   19r     FIFO               0,12      0t0     75932 pipe
ovs-vswit 21764 root   20w     FIFO               0,12      0t0     75932 pipe
ovs-vswit 21764 root   21u      CHR                1,3      0t0      1029 /dev/null

# 有一个db进程
$ ps -ef | grep ovsdb-server
root       58082       1  0 5月29 ?       00:00:00 ovsdb-server: monitoring pid 58083 (healthy)
root       58083   58082  0 5月29 ?       00:01:25 ovsdb-server /etc/openvswitch/conf.db -vconsole:emer -vsyslog:err -vfile:info --remote=punix:/var/run/openvswitch/db.sock --private-key=db:Open_vSwitch,SSL,private_key --certificate=db:Open_vSwitch,SSL,certificate --bootstrap-ca-cert=db:Open_vSwitch,SSL,ca_cert --no-chdir --log-file=/var/log/openvswitch/ovsdb-server.log --pidfile=/var/run/openvswitch/ovsdb-server.pid --detach --monitor
root     1224787 1224164  0 10:53 pts/0    00:00:00 grep ovsdb-server

```

``` bash
第1列：USER，表示进程的用户 ID。
第2列：PID，表示进程 ID。
第3列：%CPU，表示进程占用的 CPU 百分比。
第4列：%MEM，表示进程占用的内存百分比。
第5列：VSZ，表示进程占用的虚拟内存大小。
第6列：RSS，表示进程占用的物理内存大小。
第7列：TTY，表示进程所属的终端。
第8列：STAT，表示进程状态，常见的状态有：
* R：运行
* S：休眠
* Z：僵尸进程
* D：不可中断的睡眠状态
第9列：START，表示进程启动时间。
第10列：TIME，表示进程累计 CPU 时间。
第11列：COMMAND，表示进程所执行的命令。
```

### 三、架构图

当pod与pod之间通信，数据流向怎么样的，每个pod的流量是从pod对应的虚拟网卡，然后流转到网桥 bre-ext，然后经控制器，转向其他网络接口（网卡），最后转向物理网卡完成数据转发。

``` bash
# 查看当前网桥的拓扑图
$ ovs-vsctl show
```

![ovs在多台宿主机之间创建虚拟网桥网卡控制器形式转发流量](/images/ovs-1.png)
![解释什么叫做流表](/images/ovs-3.png)
![ovs在单宿主机之间数据走向标准](/images/ovs-2.png)
![ovs与内核模块之间的关系](/images/ovs-4.png)
![ovs内部模块架构图](/images/ovs-arch.png)


### 四、数据库结构和 ovs-vsctl 有2个进程

- ovsdb-server 维护数据库/etc/openvswitch/conf.db
- ovs-vswitchd 核心daemon
- 两者通过unix domain socket /var/run/openvswitch/db.sock 互相通信

ovs-vsctl 与 ovsdb-server通信，来修改数据库。ovs-vswitchd会和ovsdb-server进行通信，来对虚拟设备做相应的修改。

![数据表结构](/images/vs-db1.png)

> 通过 `cat /etc/openvswitch/conf.db` 或者 `ovsdb-client dump` 可以查看数据库表

![数据表之间的关系](/images/ovs-db2.png)

### Q&A

1. 东西向和南北向流量指的是什么

东西向指的是pod与pod之间，南北向指的是宿主机和pod之间的流量。

### 相关资料

- [set-manager主动连接ovsdb操作流解释](https://blog.csdn.net/wanglei1992721/article/details/105382332)
- [刘超的通俗云计算](https://www.cnblogs.com/popsuper1982/p/3800574.html)