---
title: docker离线安装
index_img: /images/bg/k8s.webp
banner_img: /images/bg/5.jpg
tags:
  - docker
  - linux
categories:
  - docker
date: 2023-04-18 18:40:12
excerpt: 离线安装docker engine
sticky: 1
---

### 方式一：导出rpm包或者deb包进行离线安装 - 推荐

``` bash
# rpm包导出
$ yum -y reinstall --downloadonly --downloaddir=./ docker

# 安装rpm包
$ rpm -ivh ./*.rpm
$ rpm -ivh package_name.rpm

# deb包导出
$ apt-get install dpkg-repack
$ dpkg-repack ${package-name}
```

### 方式二：下载离线包并配置systemd

1. 下载安装包

``` txt
# docker离线安装包

Docker离线安装包官方下载链接：
Docker Engine: https://docs.docker.com/engine/install/binaries/
Docker Compose: https://github.com/docker/compose/releases
Docker Machine: https://github.com/docker/machine/releases
注意：离线安装包的下载可能比在线安装包的下载时间更长，建议选择适合自己网络和设备的安装方式。
```

2. 上传到linux服务器

``` bash
# 解压
$ tar xzvf /path/to/<FILE>.tar.gz

# 移动
$ sudo cp docker/* /usr/bin/
```

3. 加入systemctl服务

``` bash
$ vim /etc/systemd/system/docker.service
```

``` shell
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target
  
[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd --selinux-enabled=false --insecure-registry=127.0.0.1
ExecReload=/bin/kill -s HUP $MAINPID
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process
# restart the docker process if it exits prematurely
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s
  
[Install]
WantedBy=multi-user.target
```

``` bash
$ chmod +x /etc/systemd/system/docker.service
```

``` bash
# 重载配置
$ systemctl daemon-reload

# 加入docker服务
systemctl enable docker

# 启动docker
systemctl start docker

# status
systemctl status docker
```


### 参考资料

[docs.docker.com/docker的二进制安装](https://docs.docker.com/engine/install/binaries/)

