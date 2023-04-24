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

### 四、验证使用kata-containerd启动容器

``` bash
$ docker run --net=none busybox uname -a
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