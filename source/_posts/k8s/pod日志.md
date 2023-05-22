---
title: 集群级日志
hide: true
excerpt: 如何查看集群级别日志，如何通过kubectl查看，并且kubelet更改存档配置，轮转配置是什么，存储路径是什么
---

https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/logging/
https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands

集群级日志其实就是pod日志，每个pod的启动退出，会往容器的标准输出写日志，这个标准输出也会被写到宿主机上的/var/log路径，这个日志可以在/var/log路径查看。

1. 应用日志(如果发生容器崩溃、Pod 被逐出或节点宕机等情况，你可能想访问应用日志)

``` bash
$ kubectl logs ${contianer_name}
$ kubectl logs counter -c count
```

> Pod 写入 40 MiB 的日志，并且 kubelet 在 10 MiB 之后轮转日志， 则运行 kubectl logs 将最多返回 10 MiB 的数据

上面这几句话一堆的我就总结出一句：kubectl logs可以查看容器日志（这个日志来源和进入容器内的查看的标准输出一致）,并且kubelet logs受限于日志轮转

如果想看其他的日志比如linux宿主机上的Systemd管理的进程的日志，那么通过journalctl去查看 

journalctl -u kubelet

如果 systemd 不存在，kubelet 和容器运行时将写入到 /var/log 目录中的 .log 文件

2. 组件

- 运行在linux上的进程，如果用systemd管理的进程，那么查看日志的方式journalctl
- Pod 中运行的 Kubernetes 集群组件，其日志会写入 /var/log 目录中的文件

3. 审计

https://kubernetes.io/zh-cn/docs/tasks/debug/debug-cluster/audit/

4. 日志解决方案之边车容器，参考 [传输数据流的边车容器](https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/logging/#%E4%BC%A0%E8%BE%93%E6%95%B0%E6%8D%AE%E6%B5%81%E7%9A%84%E8%BE%B9%E8%BD%A6%E5%AE%B9%E5%99%A8)

``` bash
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox:1.28
    - name: varlog
      mountPath: /var/log
  - name: count-log-1
    - name: varlog
      mountPath: /var/log
  - name: count-log-2
    - name: varlog
      mountPath: /var/log
  volumes:
  # 以上3个容器访问的/var/log指向同一个存储位置
  - name: varlog
    emptyDir: {}
```

5. 节点级代理 - 每个节点都有一个代理，使用daemonSet实现

- stdout 和 stderr 传输流有什么区别
- 外挂的volumn可以是一个所有节点的容器都可以访问的 namespace下面的pod一样的吗
- 外挂的volumn可以不是一个固定的宿主机上面的path对吧
- kubectl logs 查看日志原理
- kubectl logs 可以查看所有日志吗

