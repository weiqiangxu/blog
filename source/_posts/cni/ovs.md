---
title: OVS入门
index_img: /images/bg/k8s.webp
banner_img: /images/bg/5.jpg
tags:
  - kubernetes
categories:
  - kubernetes
date: 2023-04-23 18:40:12
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
[root@i-7B581709 ~]# 
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