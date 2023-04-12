---
title: alertmanager高可用机制
index_img: /images/prometheus_icon.jpeg
banner_img: /images/bg/3.jpg
tags:
  - alertmanager
  - 告警
  - 高可用
categories:
  - prometheus
date: 2023-04-12 18:40:12
---

### 一、官方推荐的高可用架构

![alertmanager集群架构图](/images/prom-ha-with-am-gossip.png)

### 二、如何保证消息不重复

``` bash
Alertmanager使用Gossip机制来解决消息在多台Alertmanager之间的传递问题。

通过在Alertmanager之间相互交流，将信息同步到所有Alertmanager的方式来避免重复发送给receiver

# 告警流程

1. 当一台Alertmanager接收到告警信息后，它会将这个信息广播给其他Alertmanager；
2. 其他Alertmanager也会将这个信息广播给其他Alertmanager，直到所有Alertmanager都收到了这个信息；
3. 其中只有一台Alertmanager会将这个告警通知发送给接收者；
```
![高可用架构示意图](/images/ha-alertmanager.png)


### 三、Gossip协议
``` txt
一种去中心化的信息传播协议

# 基本思想是
将消息从一个节点向其他节点随机传播，直到所有节点都收到了该消息，从而实现整个网络范围内的消息传递
任意节点是对等的，无固定中心节点（去中心化、高可用性、可扩展性）

# 应用场景
Gossip 协议可以应用于分布式数据库复制、分布式存储系统、大规模传感器网络等场景中
```

![Gossip协议节点通讯示意图](/images/gossip-protoctl.png)

### 四、协议在Alertmanager的应用场景

``` bash
# Slience

Alertmanager启动阶段基于Pull-based从集群其它节点同步Silence状态
新的Silence产生时使用Push-based方式在集群中传播Gossip信息

# 告警通知发送完
Push-based同步告警发送状态

# 集群alertmanager成员变化
成员发现、故障检测和成员列表更新等，简而言之，每个alertmanager都知道所有的alertmanager的存在
```

### 五、Alertmanager如何基于Gossip实现告警不重复的

1. 首先，Alertmanager节点之间通过Gossip协议建立相互联系。每个Alertmanager节点都会维护一个Gossip池，用于存储其他节点的状态信息。
2. 当集群之中某一个Alertmanager节点接到告警时，首先会计算该告警的Fingerprint（即告警内容的摘要），并将其作为ID使用。
2. 然后Alertmanager将该告警的Fingerprint广播给其他Alertmanager节点。
3. 接收到广播的Alertmanager会将该Fingerprint添加到自己的已知Fingerprint列表中。
4. 当Alertmanager要发送一个告警通知时，会检查该告警的Fingerprint是否已经存在于已知Fingerprint列表中。如果已经存在，则不会发送该告警通知。

> 也就是说每一个alertmanager节点都拥有所有的告警的Fingerprint，这个Fingerprint列表就是抑制重复发送的ID

### 六、Alertmanager的Gossip协议有哪些特点

``` bash
1. 高效：Gossip协议使用随机化的节点选择和增量式的消息传递方式，能够在短时间内将信息传递给集群中的所有节点。
2. 可靠：Gossip协议采用反馈机制和故障检测算法，能够检测并快速恢复集群中的故障节点，保证集群的稳定性和可靠性。
3. 去中心化：Gossip协议不依赖于任何中心节点或集中式控制器，所有节点都是平等的，能够自组织、自平衡和自适应。
```

### 七、Alertmanager的集群缺点

``` txt
1. 增加了复杂性：Alertmanager集群需要进行配置和管理，增加了系统复杂性。
2. 需要更多的资源：Alertmanager集群需要更多的资源，包括计算、存储和网络等。
3. 需要进行监控和日志管理：Alertmanager集群需要进行监控和日志管理，以便及时发现和解决问题。
5. 信息安全性：Alertmanager集群需要注意安全性，尤其是在跨网络或公共网络上运行时。
```

### 八、什么情况下告警仍然重复发送

``` bash
1. 集群中的某些节点出现网络故障（比如节点1和节点2同时接收到同一个告警，但是节点1和节点2之间无法通讯）
```

### 九、两台alertmanager同时接收到告警不会各自立刻发送给receiver导致重复发送吗

![alertmanager pipeline](/images/alertmanager-HA-arch.png)

> 当一个Alertmanager实例将告警通知发送给Receiver时，它会将该通知标记为“暂停发送”，同时向其他实例发送消息，告诉它们“我已经发送了这个告警通知，你们不用再发送了”。这种方式可以确保告警通知在多个实例之间被正确地合并，避免重复发送。剩余多个Alertmanager实例同时接收到相同的告警信息，并且它们之间的通信还没有完成，那么它们都会标记该告警信息为“暂停发送”，并在通信完成之后再决定由哪个实例发送该告警通知。因此，在Alertmanager中，重复发送同一告警通知的情况应该是非常少见的。

<!-- ``` bash
# 流程图解读

1. 某个Alertmanager接收到告警，会等待一段时间（默认为30秒）看其他Alertmanager节点是否接到该告警
2. 等待聚合时间结束后，Alertmanager集群仍然没有收到该告警的聚合，则它将发送给接收器（receiver）
等待时间称为“等待聚合”（wait_for_aggregate）时间，可以在Alertmanager配置文件的route部分进行配置;
``` -->

<!-- ![alertmanager接收到告警之后如何处置](/images/am-notifi-pipeline.png) -->

<!-- ![prometheus发送给alertmanager后告警流程](/images/am-gossip.png) -->



### alertmanager集群只有最早接收到告警alertmanager节点才会发送给接收器是吗，其他节点不会发送给接收器是是吗



### docker搭建alertmanager集群

``` yml
# 集群模式下 alertmanager 绑定的 IP 地址和端口号
# 用于与其他 alertmanager 节点进行通信。
# 通过 cluster.listen-address 监听并接收来自其他 alertmanager 节点的请求，并将自己的状态信息同步给其他节点
cluster.listen-address

# alertmanager的cluster.peer是指alertmanager节点在集群中的对等节点
# 用于配置alertmanager的高可用性，确保即使某个节点出现问题，其他节点也能够继续工作
# 当alertmanager节点加入到集群中时，它会将自己的peer信息发送给其他节点，其他节点也会将自己的peer信息发送给它
# 这样，每个alertmanager节点就可以知道其他节点的状态，并在需要时进行切换和故障转移。
#
# 实例的cluster.peer参数，以指定其对等节点的地址
cluster.peer
```

``` bash
# 创建alertmanager的配置
# 接收到告警之后甩给webhook

$ touch /Users/xuweiqiang/Desktop/a1.yml
$ touch /Users/xuweiqiang/Desktop/a2.yml
$ touch /Users/xuweiqiang/Desktop/a3.yml
```

``` yml
# 配置alertmanager向webhook发送告警信息
route:
  receiver: 'default-receiver'
receivers:
  - name: default-receiver
    webhook_configs:
    - url: 'http://docker.for.mac.host.internal:5001/'
```

``` bash
# a1
$ docker run -d \
    --network p_net \
    --network-alias a1 \
    --name=a1 \
    -p 9093:9093 \
    -v /Users/xuweiqiang/Desktop/a1.yml:/etc/alertmanager/config.yml \
    prom/alertmanager:latest \
    --web.listen-address=":9093" \
    --cluster.listen-address=":8001" \
    --config.file=/etc/alertmanager/config.yml \
    --log.level=debug
```

[http://localhost:9093/#/status](http://localhost:9093/#/status)

``` bash
# a2
$ docker run -d \
    --network p_net \
    --network-alias a2 \
    --name=a2 \
    -p 9094:9094 \
    -v /Users/xuweiqiang/Desktop/a2.yml:/etc/alertmanager/config.yml \
    prom/alertmanager:latest \
    --web.listen-address=":9094" \
    --cluster.listen-address=":8001" \
    --cluster.peer=a1:8001 \
    --config.file=/etc/alertmanager/config.yml \
    --log.level=debug
```

[http://localhost:9094/#/status](http://localhost:9094/#/status)

``` bash
# a3
$ docker run -d \
    --network p_net \
    --network-alias a3 \
    --name=a3 \
    -p 9095:9095 \
    -v /Users/xuweiqiang/Desktop/a3.yml:/etc/alertmanager/config.yml \
    prom/alertmanager:latest \
    --web.listen-address=":9095" \
    --cluster.listen-address=":8001" \
    --cluster.peer=a1:8001 \
    --config.file=/etc/alertmanager/config.yml \
    --log.level=debug
```

[http://localhost:9095/#/status](http://localhost:9095/#/status)

``` bash
# webhook
$ go install github.com/prometheus/alertmanager/examples/webhook
# 启动服务
$ webhook
```

### 测试发送告警

``` bash
alerts1='[
  {
    "labels": {
       "alertname": "DiskRunningFull",
       "instance": "example1"
     },
     "annotations": {
        "info": "The disk sdb1 is running full",
        "summary": "please check the instance example1"
      }
  }
]'

curl -XPOST -d"$alerts1" http://localhost:9093/api/v1/alerts
curl -XPOST -d"$alerts1" http://localhost:9094/api/v1/alerts
curl -XPOST -d"$alerts1" http://localhost:9095/api/v1/alerts
```

### prometheus集群与alertmanager集群

``` yml
# 总的一句话就是：每个prometheus往alertmanager集群的所有机器发送告警
# 最大限度保证告警消息不会丢失就好了
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - 127.0.0.1:9093
      - 127.0.0.1:9094
      - 127.0.0.1:9095
```

### 0.0.0.0和localhost和127.0.0.1之间什么关系

``` txt
0.0.0.0、localhost和127.0.0.1都是指向本地主机的IP地址，但是它们之间有所区别。

- 0.0.0.0是一种未指定特定网络地址的地址，在网络编程中通常用于表示所有的IP地址（包括本地主机和其他主机）
- localhost是一个特殊的域名，它指向本地主机（即127.0.0.1），通常用于测试和开发等目的
- 127.0.0.1是本机回环地址，即指向本地主机的IP地址，通常用于本地通信和测试等目的

因此，这三者都是指向本地主机的地址，但是使用场景和用途略有区别
```

### 参考资料

[Alertmanager高可用](https://yunlzheng.gitbook.io/prometheus-book/part-ii-prometheus-jin-jie/readmd/alertmanager-high-availability#chuang-jian-alertmanager-ji-qun)

[https://github.com/prometheus/alertmanager](https://github.com/prometheus/alertmanager)