---
title: containerd和kata&firecracker环境搭建
index_img: /images/bg/k8s.webp
banner_img: /images/bg/5.jpg
tags:
  - kubernetes
categories:
  - kubernetes
date: 2023-04-23 18:40:12
excerpt: 安装containerd和kata，测试kata-qemu和kata-firecracker
sticky: 1
---

### 一、环境

``` bash
[root@VM-8-4-centos ~]# uname -a
Linux x86_64 GNU/Linux

# 需要支持虚拟化：有输出表示支持
$ egrep '(vmx|svm)' /proc/cpuinfo |wc -l

# 需要安装kvm
$ lsmod | grep kvm
# kvm_intel  376832  11
# kvm  1015808  1 kvm_intel
```

### 二、安装包

``` txt
kata-static-3.0.2-x86_64.tar.xz
```

### 三、文档地址

- [kata-containers/3.0.2-如何与containerd集成](https://github.com/kata-containers/kata-containers/blob/3.0.2/docs/how-to/containerd-kata.md)
- [containerd-v1.7.0安装snapshotters.devmapper](https://github.com/containerd/containerd/blob/v1.7.0/docs/snapshotters/devmapper.md)
- [kata-containerd/v3.0.2](https://github.com/kata-containers/kata-containers/releases/tag/3.0.2)
- [kata-container和containerd安装](https://github.com/kata-containers/kata-containers/blob/main/docs/install/container-manager/containerd/containerd-install.md)


### 四、安装kata-containers

``` bash
# 下载安装包
$ /home/kata-static-3.0.2-x86_64.tar.xz

# 解压至根目录
$ tar -xvf  kata-static-3.0.2-x86_64.tar.xz -C /

# 验证可用
$ kata-runtime check --no-network-checks
$ kata-runtime --show-default-config-paths
$ kata-runtime kata-env
```

### 五、配置containerd

``` yml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata]
  runtime_type = "io.containerd.kata.v2"
  privileged_without_host_devices = true
  pod_annotations = ["io.katacontainers.*"]
  container_annotations = ["io.katacontainers.*"]
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata.options]
    ConfigPath = "/opt/kata/share/defaults/kata-containers/configuration.toml"
```
[containerd.plugins.cri.runtimes.kata配置说明](https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/containerd-kata.md#kata-containers-as-a-runtimeclass)

``` bash
# 重启containerd服务
$ systemctl daemon-reload
$ systemctl start containerd 
```

``` bash 
$ containerd config dump | grep kata
```

### 六、运行容器

``` bash
$ sudo ctr image pull docker.io/library/busybox:latest
$ sudo ctr run --runtime "io.containerd.kata.v2" --rm -t docker.io/library/busybox:latest test-kata uname -r
```

### 七、使用firecracker创建容器


[how-to-use-kata-containers-with-firecracker](https://github.com/kata-containers/kata-containers/blob/3.0.2/docs/how-to/how-to-use-kata-containers-with-firecracker.md)

``` bash
# devmapper非常重要
# devmapper非常重要
$ sudo ctr plugins ls | grep devmapper

# 创建符号连接否则containerd找不到kata
$ sudo ln -s /home/opt/kata/bin/containerd-shim-kata-v2 /usr/bin/containerd-shim-kata-v2
```

``` bash
# containerd.plugin.devmapper需要安装
$ sudo ctr images pull --snapshotter devmapper docker.io/library/ubuntu:latest
$ sudo ctr run --snapshotter devmapper --runtime io.containerd.run.kata-fc.v2 -t --rm docker.io/library/ubuntu
```

### Q&A

- kata-rc怎么和containerd集成

``` txt
kata runtime独立仓库(v1.5) 之前出的一个兼容fc的脚本
新版本3.0需要了
```

- kata-runtime和kata-containerd什么关系

```
kata-container 包含 kata-runtime
```

- containerd怎么集成kata-rc
- containerd怎么安装扩展plugins.devmapper
- containerd刚刚安装的时候没有配置文件怎么生成
```
containerd config default > /etc/containerd/config.toml
```
- kata-runtime刚刚生成没有配置文件怎么处理
- containerd 怎么添加扩展
- containerd的devmapper是什么来的
- CNI怎么安装，etc/cni/net.d/这个文件夹下面的配置是怎么填写的

- rootfs not found
[https://github.com/kata-containers/kata-containers/issues/6784](https://github.com/kata-containers/kata-containers/issues/6784)

- kata container amd64下载
[https://github.com/kata-containers/kata-containers/issues/6776](https://github.com/kata-containers/kata-containers/issues/6776)


### 相关资料

[kata-firecracker和docker的集成](https://github.com/kata-containers/documentation/wiki/Initial-release-of-Kata-Containers-with-Firecracker-support)