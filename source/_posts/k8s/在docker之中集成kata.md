---
title: 在docker之中集成kata
index_img: /images/bg/k8s.webp
banner_img: /images/bg/5.jpg
tags:
  - docker
  - kata
categories:
  - kubernetes
date: 2023-04-18 18:40:12
excerpt: 如何在docker之中使用kata
sticky: 1
---


### 一、离线安装Docker

[离线安装docker](https://weiqiangxu.github.io/2023/04/18/kubernetes/%E8%AF%AD%E9%9B%80k8s%E5%9F%BA%E7%A1%80%E5%85%A5%E9%97%A8/docker%E7%A6%BB%E7%BA%BF%E5%AE%89%E8%A3%85/)

> https://download.docker.com/linux/static/stable/aarch64/docker-23.0.4.tgz

### 二、kata-containers安装

1. 检测是否支持kata-containers

``` bash
# 先查看当前架构
$ uname -a

# x86和AMD判断支持Intel VT-x, AMD SVM
# Intel的处理器支持Intel VT-x技术，而AMD的处理器支持AMD SVM技术
# KVM虚拟化需要处理器支持Intel VT-x或AMD SVM
# 要看cpu型号是否支持虚拟化
# 判断是否支持kvm可以知道当前系统是否支持虚拟化
# 对于Intel处理器(如果返回值大于0，则表示处理器支持Intel VT-x，并且操作系统在启动时已启用虚拟化支持)
$ grep -cw vmx /proc/cpuinfo
$ egrep -c '(svm|vmx)' /proc/cpuinfo
$ cat /proc/cpuinfo | grep -E "svm|vmx"  # x86平台

# 对于AMD处理器(如果返回值大于0，则表示处理器支持AMD SVM，并且操作系统在启动时已启用虚拟化支持)
$ grep -cw svm /proc/cpuinfo
$ cat /proc/cpuinfo

# 检查KVM模块是否安装
$ lsmod | grep kvm

# aarch64判断支持ARM Hyp
$ grep -i "HYP" /proc/cpuinfo
$ cat /proc/cpuinfo | grep "hypervisor"  # ARM平台

# mac判断支持虚拟化
$ sysctl kern.hv_support
```

2. 安装教程

[kata-containers/docs/install](https://github.com/kata-containers/kata-containers/tree/main/docs/install)

### 三、配置docker使用kata-containers

``` bash
# 第一种方式：systemd
$ mkdir -p /etc/systemd/system/docker.service.d/

$ cat <<EOF | sudo tee /etc/systemd/system/docker.service.d/kata-containers.conf
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -D --add-runtime kata-runtime=/usr/bin/kata-runtime --default-runtime=kata-runtime
EOF
```

``` bash
# 第二种方式：daemon.json
$ vim /etc/docker/daemon.json
```

``` yml
{
  "default-runtime": "kata-runtime",
  "runtimes": {
    "kata-runtime": {
      "path": "/usr/bin/kata-runtime"
    }
  }
}
```

``` bash
# 查看docker支持的runtime有哪些
$ docker info | grep runtime
```

### 四、验证使用kata-containerd启动容器

``` bash
# 基于kata-runtime执行
$ docker run --net=none -itd --name centos-test centos:centos7

# 基于runc执行
$ docker run --net=none --runtime=runc -itd --name centos-test1 centos:centos7

# 进入容器之中
$ docker exec -it centos-test1  /bin/bash
$ docker exec -it centos-test  /bin/bash

# 运行时为runc的内存和运行时为kata-runtime的是不一样的
# 运行时为runc的内存和宿主机上直接的内存是一样的
$ free -m

```


### 疑问

- OCI runtime create failed: QEMU path (/usr/bin/qemu-kvm) does not exist

``` bash
$ yum install -y qemu-kvm
```
-  messages from qemu log: Could not access KVM kernel module: No such file or directory

``` bash
f2进入bios界面，查找virtual字样的选项，将其开启(enable)
```

- sandbox interface because it conflicts with existing route

```
--net=none
```

- 容器安全

使用Docker轻量级的容器时，最大的问题就是会碰到安全性的问题，其中几个不同的容器可以互相的进行攻击，如果把这个内核给攻掉了，其他所有容器都会崩溃。
如果使用KVM等虚拟化技术，会完美解决安全性的问题，但响应的性能会受到一定的影响。
单单就Docker来说，安全性可以概括为两点： - 不会对主机造成影响 - 不会对其他容器造成影响所以安全性问题90%以上可以归结为隔离性问题。
而Docker的安全问题本质上就是容器技术的安全性问题，这包括共用内核问题以及Namespace还不够完善的限制： 
1. /proc、/sys等未完全隔离 
2. Top, free, iostat等命令展示的信息未隔离 
3. Root用户未隔离 
4. /dev设备未隔离 
5. 内核模块未隔离 
6. SELinux、time、syslog等所有现有Namespace之外的信息都未隔离

### 相关链接

[什么是容器安全](https://zhuanlan.zhihu.com/p/109256949)