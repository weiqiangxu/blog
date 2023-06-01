---
title: Helm使用手册
index_img: /images/bg/k8s.webp
banner_img: /images/bg/5.jpg
tags:
  - kubernetes
categories:
  - kubernetes
date: 2023-06-01 16:28:12
excerpt: 介绍如何使用image镜像制作helm包、如何更新、卸载，在k8s上的资源如何使用helm更好的维护
sticky: 1
hide: false
---

### 一、Helm相关概念

Helm是一个kubernetes的包管理工具，软件包管理器 (类似ubuntu.apt centos.yum python.pip)，主要解决Kubernetes原生资源文件如deployment、replicationcontroller、service或pod等资源过于分散不好管理的问题。

- helm 命令行管理工具，kubernetes应用chart（创建、打包、发布和管理）
- chart 应用描述（kubernetes资源相关文件的集合），相当于是k8s的Yum RPM
- release 基于chart的部署实体，chart被Helm运行后生成一个release,比如MySQL chart可以有很多个实例（每一次安装有它自己的 release && release name）
- Repository 存放和共享 charts 的地方

[Helm QuickStart](https://helm.sh/zh/docs/intro/using_helm/)

### 二、安装方式

1. 使用脚本安装
2. 用二进制版本安装(tar -zxvf)

### 三、常用命令

``` bash
# 添加私有仓库
$ helm repo add bingokube http://registry.company.net/chartrepo/company

$ helm repo update

# 从站点 https://artifacthub.io/ 寻找 chart 包
$ helm search hub xxx

# 从你添加（使用 helm repo add）到本地 helm 客户端中的仓库中进行查找
$ helm search repo xxx

# 示例查找 MySQL 包
$ helm search hub mysql

# https://artifacthub.io/packages/helm/mysql/mysql
# NAME: happy-panda
$ helm install happy-panda bitnami/wordpress

# 追踪release状态
$ helm status happy-panda

# 查看chart包配置选项
$ helm show values bitnami/wordpress

# 卸载
$ helm uninstall happy-panda

# 升级
$ helm upgrade -f panda.yaml happy-panda bitnami/wordpress

# 回滚
$ helm rollback ${release} ${reversion}
```

### 四、制作部署loki&promtail的helm包


### 相关资料

[自定义chart](https://helm.sh/zh/docs/intro/using_helm/#%E5%AE%89%E8%A3%85%E5%89%8D%E8%87%AA%E5%AE%9A%E4%B9%89-chart)
[Helm 命令集合](https://helm.sh/zh/docs/helm/helm/)
[chart 语法大全](https://helm.sh/zh/docs/chart_template_guide/getting_started/)
[helm究竟是什么](https://bbs.huaweicloud.com/blogs/280351)