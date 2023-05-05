### 环境

```
[root@VM-8-4-centos ~]# uname -a
Linux VM-8-4-centos 3.10.0-1160.71.1.el7.x86_64 #1 SMP Tue Jun 28 15:37:28 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux
```

### 安装包

``` txt
kata-static-3.1.0-x86_64.tar.xz
cni-plugins-linux-amd64-v1.2.0.tgz
runc.amd64
containerd.service
```

### 具体步骤

[下载kata 3.1.0](https://github.com/kata-containers/kata-containers/blob/main/docs/install/container-manager/containerd/containerd-install.md)

[containerd安装详细](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)

``` sh
# containerd
$ tar xvf containerd-1.7.0-linux-amd64.tar.gz
$ mv bin/* /usr/local/bin/

# containerd system service
$ touch /usr/local/lib/systemd/system/containerd.service
$ wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
$ systemctl daemon-reload
$ systemctl enable --now containerd

# runc
$ install -m 755 runc.amd64 /usr/local/sbin/runc

# cni
$ cni 
$ systemctl status containerd
```



### kata

``` bash
$ cd /home && tar xvf kata-static-3.1.0-x86_64.tar.xz
$ vi ~/.bashrc
# 添加一行
# export PATH=$PATH:/home/opt/kata/bin
$ source ~/.bashrc
$ kata-runtime --version
# 检测是否可以运行
$ kata-runtime kata-check
```

### 更改containerd配置将kata集成到containerd之中

``` sh
# 查看默认配置
$ containerd config default

# 查看当前配置如果没有会提示没有配置文件
$ containerd config dump

# 没有配置文件就自己手动创建一个
$ mkdir -p /etc/containerd
$ vim /etc/containerd/config.toml
$ containerd config dump
```

``` yml
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    [plugins."io.containerd.grpc.v1.cri".containerd]
      default_runtime_name = "kata"
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata]
          runtime_type = "io.containerd.kata.v2"
```

``` bash
# 创建符号连接否则containerd找不到kata
$ sudo ln -s /home/opt/kata/bin/containerd-shim-kata-v2 /usr/bin/containerd-shim-kata-v2
```

``` bash
$ cat /etc/containerd/config.toml | grep kata
```

# 测试验证


``` sh
$ image="docker.io/library/busybox:latest"
$ sudo ctr image pull "$image"
$ sudo ctr run --runtime "io.containerd.kata.v2" --rm -t "$image" test-kata uname -r
```



[kata-firecracker和docker的集成](https://github.com/kata-containers/documentation/wiki/Initial-release-of-Kata-Containers-with-Firecracker-support)


1. kata-rc怎么和containerd集成
2. kata-runtime和kata-containerd什么关系
3. containerd怎么集成kata-rc
4. containerd怎么安装扩展plugins.devmapper
5. containerd刚刚安装的时候没有配置文件怎么生成
```
containerd config default > /etc/containerd/config.toml.
```


6. kata-runtime刚刚生成没有配置文件怎么处理
7. containerd 怎么添加扩展