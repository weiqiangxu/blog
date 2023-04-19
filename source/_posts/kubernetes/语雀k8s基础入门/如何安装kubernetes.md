---
title: 如何安装kubernetes
index_img: /images/bg/k8s.webp
banner_img: /images/bg/5.jpg
tags:
  - kubernetes
categories:
  - kubernetes
date: 2023-04-18 18:40:12
excerpt: 使用kubeadm搭建kubenertes环境
sticky: 1
---

### 一、环境准备

CentOS 7.6 64bit 2核 2G * 2 ( 操作系统的版本至少在7.5以上 )


#### 1.防火墙

``` bash
$ systemctl stop firewalld
$ systemctl disable firewalld
```

#### 2.设置主机名

``` bash
# 在master节点执行
$ hostnamectl set-hostname k8s-master

# 在其他节点执行
$ hostnamectl set-hostname k8s-node1
```

#### 3.主机域名解析

``` bash
# 注意：这里的IP地址是机器的局域网IP地址
$ cat >> /etc/hosts << EOF
10.0.20.2 k8s-master
EOF
```

#### 4.统一时间

``` bash
$ yum install ntpdate -y
$ ntpdate time.windows.com
```

#### 5.关闭selinux

``` bash
#查看selinux是否开启
$ getenforce
```

``` bash
# 永久关闭selinux，需要重启
# 使用sed工具在/etc/selinux/config文件中查找包含“enforcing”的行并将其替换为“disabled”
$ sed -i 's/enforcing/disabled/' /etc/selinux/config
```

#### 6.关闭swap分区

``` bash
# 永久关闭swap分区，需要重启：
# 使用sed工具在/etc/fstab文件中查找任何包含“swap”的行
# 并在每行前加上“#”注释符，从而将它们全部注释掉
$ sed -ri 's/.*swap.*/#&/' /etc/fstab
```

#### 7.将桥接的IPv4流量传递到iptables的链

``` bash
#在每个节点上将桥接的IPv4流量传递到iptables的链
$ cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

# 重新加载系统参数并应用任何更改
# 通常用于修改系统级别的参数，例如内核参数、网络配置等
$ sysctl --system
```

### 二、docker和二进制程序


#### 1.安装docker

``` bash
# 从阿里云的镜像站点上下载docker-ce的yum源配置文件docker-ce.repo
# 并将其保存到本地的/etc/yum.repos.d/目录下
$ wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo

# 各节点安装docker
$ yum -y install docker-ce-18.06.3.ce-3.el7

# docker添加到系统服务并启动docker
$ systemctl enable docker && systemctl start docker

# 验证安装结果
$ docker version
```

#### 2.设置Docker镜像加速器：

``` bash
# 创建docker配置
$ sudo mkdir -p /etc/docker

# 设置docker国内镜像
# tee: 该命令用于向文件中写入内容，并在标准输出中显示相同的内容
# 将大括号的空JSON对象写入到/etc/docker/daemon.json文件中
$ sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "exec-opts": ["native.cgroupdriver=systemd"], 
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"], 
  "live-restore": true,
  "log-driver":"json-file",
  "log-opts": {"max-size":"500m", "max-file":"3"},
  "storage-driver": "overlay2"
}
EOF

# systemctl重载服务配置文件
$ sudo systemctl daemon-reload

# 重启docker服务
$ sudo systemctl restart docker
```

#### 3.kubernetes镜像源

``` bash
# 注意下面的mirrors是区分架构的
# 查看当前机器的架构
$ arch

# 访问地址 https://mirrors.aliyun.com/kubernetes/yum/repos 获取所有架构镜像源
# 更改下面的 baseurl
```

``` bash
# 注意执行之前确保baseurl后面的镜像源架构是否匹配
$ cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

#### 4.安装kubeadm\kubelet\kubectl

``` bash
# install
$ yum install -y kubelet-1.18.0 kubeadm-1.18.0 kubectl-1.18.0

# 启用kubelet服务,开机自启动
$ systemctl enable kubelet

# 移除kubelet的CNI网络插件设置
# 文件`/var/lib/kubelet/kubeadm-flags.env`中
# 将所有的`--network-plugin=cni`字符串替换为空字符串
$ sed -i 's/--network-plugin=cni//' /var/lib/kubelet/kubeadm-flags.env
$ systemctl restart kubelet

# 配置在Kubernetes集群中使用Flannel网络插件时的CNI插件参数
# Flannel是一种软件定义网络（SDN）解决方案，它使用虚拟网络来连接容器和节点
# 需要在集群中的每个节点上配置Flannel CNI插件参数
# 以便容器运行环境能够正确使用Flannel网络插件
$ mkdir -p /etc/cni/net.d
$ cat > /etc/cni/net.d/10-flannel.conf <<EOF
{
  "name": "cbr0",
  "cniVersion": "0.2.0",
  "type": "flannel",
  "delegate": {
    "isDefaultGateway": true
  }
} 
EOF
```

#### 5.安装CNI插件

``` bash
# https://github.com/containernetworking/plugins/releases
# 进入下载页根据架构选择安装包
$ arch
$ aarch64

# AArch64是一种ARMv8架构
$ wget https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-arm64-v1.2.0.tgz

# 解压后移动bin,注意这个插件很重要
$ mv /home/cni-plugins-linux-arm64-v1.2.0/* /opt/cni/bin/
```

#### 6.kubeadm初始化集群

``` bash
# 查看k8s所需的镜像
$ kubeadm config images list

# 部署k8s的master节点
# apiserver-advertise-address更改为部署master的节点的局域网IP地址
# pod-network-cidr 集群中Pod的IP地址段 常用的有10.244.0.0/16
# service-cidr.集群中Service的IP地址段.默认为10.96.0.0/12
kubeadm init \
  --apiserver-advertise-address=192.168.18.100 \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.18.0 \
  --service-cidr=10.96.0.0/12 \
  --pod-network-cidr=10.244.0.0/16
```

``` bash
# 创建成功输出
$ Your Kubernetes control-plane has initialized successfully!

$ kubeadm join 192.168.1.1:6443 --token xxx.xx \
    --discovery-token-ca-cert-hash sha256:xxx
```

#### 7.配置kubectl环境

``` bash
# 配置信息指定Kubernetes API的访问地址、认证信息、命名空间、资源配额以及其他配置参数
$ mkdir -p $HOME/.kube

# 将admin.conf复制到当前用户的$HOME/.kube/config
# kubectl命令在使用时默认使用该文件
# 否则使用kubectl命令时必须明确指定配置文件路径或设置环境变量
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

# 将配置文件的所有权赋予给当前用户
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 三、部署集群中Flannel网络插件

``` bash
# 查看节点状态
$ kubectl get nodes

# 使用kubectl工具将kube-flannel.yml文件中定义的Flannel网络插件之中
# Deployment、DaemonSet、ServiceAccount等Kubernetes资源部署到集群中
# 从而为集群中的宿主机和容器、容器间提供网络互联功能
$ wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
$ kubectl apply -f kube-flannel.yml

# 查看集群pods确认是否成功
$ kubectl get pods -n kube-system

# 查看集群健康状况(显示了控制平面组件
# Control Plane Component）的健康状态
# etcd/kube-apiserver/kube-controller-manager/kube-scheduler
$ kubectl get cs

# 显示集群的相关信息(DNS\证书\密钥等)
$ kubectl cluster-info
```

### 四、部署nginx服务测试集群

``` bash
# 部署Nginx
$ kubectl create deployment nginx --image=nginx:1.14-alpine

# 暴露端口
$ kubectl expose deployment nginx --port=80 --type=NodePort

# 查看服务状态需要看到nginx服务为running
$ kubectl get pods,svc

# 访问nginx服务需要看到输出Welcome to nginx!
$ curl k8s-master:30185
```

### 五、其他节点部署服务并加入到集群之中

``` bash
# worker node 安装Docker/kubeadm/kubelet/kubectl
# master节点生成token(2小时过期)
$ kubeadm token create --print-join-command

# master节点生成一个永不过期的token
$ kubeadm token create --ttl 0 --print-join-command

# 在worker node执行
# --token：用于新节点加入集群的令牌
# --discovery-token-ca-cert-hash：用于验证令牌的证书哈希值(集群初始化生成)
# --control-plane：如果要将新节点添加到控制平面，则需要指定此选项
# --node-name：指定新节点的名称
$ kubeadm join 192.168.18.100:6443 \
    --token xxx \
    --discovery-token-ca-cert-hash sha256:xxx
```

### 六、相关疑问

- kubeadmn如何重新初始化

``` bash
$ kubeadm reset
```

- 查看pod详情

``` bash
$ kubectl describe pod $pod_name
$ kubectl describe pod nginx-deployment-xxx
$ kubectl describe pod mongodb -n NamespaceName
```

- run/flannel/subnet.env无法找到

``` bash
# 当错误是/run/flannel/subnet.env无法找到时候手动创建subnet.env内容是
$ FLANNEL_NETWORK=10.244.0.0/16
$ FLANNEL_SUBNET=10.244.0.1/24
$ FLANNEL_MTU=1450
$ FLANNEL_IPMASQ=true
```

- docker离线安装包

``` txt
Docker离线安装包官方下载链接：
Docker Engine: https://docs.docker.com/engine/install/binaries/
Docker Compose: https://github.com/docker/compose/releases
Docker Machine: https://github.com/docker/machine/releases
注意：离线安装包的下载可能比在线安装包的下载时间更长，建议选择适合自己网络和设备的安装方式。
```

- 路径的docker.repo是干嘛的

``` bash
$ cd /etc/yum.repos.d/docker.repo 

# etc/yum.repos.d/docker.repo 是一个yum源配置文件，用于添加docker的官方仓库到yum源中。
# 通过该文件，可以使用yum命令在CentOS/RHEL系统中安装、更新、卸载docker软件包
```

- Failed to download metadata for repo ‘docker-ce-stable

``` bash
$ cd /etc/yum.repos.d
$ rm -rf docker*
$ yum update
$ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

- yum-config-manager --add-repo干嘛用的

```
yum-config-manager --add-repo用于添加一个新的yum软件源地址，并将其保存在yum配置文件中
以便yum命令可以从该软件源中下载和安装软件包。该命令可以帮助用户获得更多的软件源
以便在软件包依赖项解决方案中更好地满足其需求。
```

- kuernetes.repo

``` bash
# repo地址
cat /etc/yum.repos.d/kubernetes.repo
```

``` bash
cat << EOF > /etc/yum.repos.d/kubernetes.repo 
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

- kube-flannel是干嘛的

``` txt
kube-flannel是一个CNI插件，用于为Kubernetes集群创建和管理网络。
它提供了一种简单而有效的方法，让容器在不同节点上进行通信，而无需手动配置网络。
kube-flannel使用VXLAN或UDP封装技术来创建一个覆盖整个集群的扁平网络，
使得Kubernetes Pod能够互相通信，同时确保网络性能的高效和可靠性。
```

- k8s的控制平面有哪些

1. kube-apiserver：Kubernetes API服务器，提供Kubernetes API的访问入口，以及对Kubernetes内部对象的认证、授权和验证。
2. etcd：Kubernetes使用etcd作为其默认的分布式键值存储系统，用于存储Kubernetes集群的所有配置数据、元数据和状态信息。
3. kube-scheduler：负责将新创建的Pod调度到集群中的Node上，选择最佳的Node来运行Pod。
4. kube-controller-manager：Kubernetes控制器管理器，包含了多个控制器，用于自动化管理Kubernetes集群中的各种资源和对象。
5. cloud-controller-manager：云控制器管理器，负责管理云平台上的资源，如EC2、ELB等，并将这些资源与Kubernetes集群进行集成。

> 这些组件共同组成了Kubernetes控制平面，负责管理和控制整个Kubernetes集群的运行状态。

- kubectl get svc的输出解释

``` bash
$ NAME：Service名称。
$ TYPE：Service类型。通常有ClusterIP、NodePort、LoadBalancer和ExternalName四种类型。
$ CLUSTER-IP：Service的Cluster IP地址。
$ EXTERNAL-IP：Service的外部IP地址（仅适用于LoadBalancer类型）。
$ PORT(S)：Service暴露的端口号和协议。
$ AGE：Service被创建后的时间。

例如，以下是一段`kubectl get svc`命令的输出结果：

NAME           TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
my-service     ClusterIP      10.0.0.5        <none>          80/TCP           5d
app-service    LoadBalancer   10.0.0.15       203.0.113.10   80:30000/TCP     2d

解释：

$ `my-service`是一个ClusterIP类型的Service，其Cluster IP地址为`10.0.0.5`，暴露的端口号为80/TCP。
$ `app-service`是一个LoadBalancer类型的Service，其Cluster IP地址为`10.0.0.15`
    暴露的端口号为80/TCP，同时外部IP地址为`203.0.113.10`，将请求转发到NodePort `30000`上
```

- linux怎么样验证一个端口通不通

``` bash
# telnet
$ telnet ${IP地址} ${端口号}
$ telnet 192.168.1.10 80

# success
$ Connected to $ip
```

- telnet 127.0.0.1 30022 出现Connection refused

``` bash
# 没有运行正在监听30022端口的服务或防火墙等安全设置阻止了连接
$ sudo firewall-cmd --state
$ sudo firewall-cmd --list-ports
```

- 只有master节点无法调度怎么办

``` bash
# 获取节点信息
$ kubuctl get node

# 查看当前mster节点所有taint
# 输出之中所有的Taints:node.kubernetes.io/not-ready:NoExecute...
$ kubectl describe node

# 取消标记语法
$ kubectl taint nodes <master-node-name> node-role.kubernetes.io/master:NoSchedule-

# 取消2个标记
$ kubectl taint nodes k8s-master node.kubernetes.io/not-ready:NoExecute-
$ kubectl taint nodes k8s-master node.kubernetes.io/not-ready:NoSchedule-
$ kubectl taint nodes k8s-master node-role.kubernetes.io/master:NoSchedule-
```

- k8s会默认设置Taints在主节点吗

```  bash
# 给主节点打上一个Key为`node-role.kubernetes.io/master`，value为`NoSchedule`的Taint，即在主节点上设置Taints
# kubectl taint nodes `master-node-name` node-role.kubernetes.io/master=:NoSchedule

默认情况下，Kubernetes会在主节点上设置Taints。这是为了确保主节点不被普通的Pod调度和运行。
只有具有对应Tolerations的Pod才能被调度和运行在主节点上。
这可以确保主节点保持稳定和安全，防止普通的Pod对主节点产生负面影响。
```

### 相关资料

[kubernetes.io/zh-cn/安装kubeadm](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
[官方docker离线安装](https://download.docker.com/linux/static/stable)
[kubernetes/yum/repos各个架构下的](https://mirrors.aliyun.com/kubernetes/yum/repos/)
[zhihu/k8s 1.16.0 版本的coreDNS一直处于pending状态的解决方案](https://zhuanlan.zhihu.com/p/602370492)
[k8s部署flannel时报failed to find plugin /opt/cni/bin](https://blog.csdn.net/qq_41586875/article/details/124688043)