---
title: alertmanager高可用机制
index_img: /images/prometheus_icon.jpeg
banner_img: /images/bg/1.jpg
tags:
  - alertmanager
  - 告警
  - 高可用
categories:
  - prometheus
date: 2023-04-11 18:40:12
---

### 多台alertmanager如何保证消息不重复

Alertmanager使用Gossip机制来解决消息在多台Alertmanager之间的传递问题。Gossip机制可以确保在多个Alertmanager之间传递相同的告警信息时，只会有一个告警通知被发送给接收者，而不会出现重复通知的情况。

简单来说，Gossip机制是通过在Alertmanager之间不断进行相互交流，将信息同步到所有Alertmanager的方式来确保消息的一致性。当一台Alertmanager接收到告警信息后，它会将这个信息广播给其他Alertmanager，其他Alertmanager也会将这个信息广播给其他Alertmanager，直到所有Alertmanager都收到了这个信息。

当所有Alertmanager都确认收到了相同的告警信息之后，只有其中一台Alertmanager会将这个告警通知发送给接收者，其他Alertmanager则不会再次发送相同的通知。

通过使用Gossip机制，可以确保在多个Alertmanager之间收到相同的告警信息时，只会有一个告警通知被发送给接收者，避免了消息的重复通知问题。

![alertmanager集群架构图](/images/prom-ha-with-am-gossip.png)

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

### Gossip协议
``` txt
一种去中心化的信息传播协议

# 基本思想是
将消息从一个节点向其他节点随机传播，直到所有节点都收到了该消息，从而实现整个网络范围内的消息传递
任意节点是对等的，无固定中心节点（去中心化、高可用性、可扩展性）

# 应用场景
Gossip 协议可以应用于分布式数据库复制、分布式存储系统、大规模传感器网络等场景中
```

![Gossip协议节点通讯示意图](/images/gossip-protoctl.png)

### 协议在Alertmanager的应用场景

``` bash
# Slience

Alertmanager启动阶段基于Pull-based从集群其它节点同步Silence状态
新的Silence产生时使用Push-based方式在集群中传播Gossip信息

# 告警通知发送完
Push-based同步告警发送状态

# 集群alertmanager成员变化
成员发现、故障检测和成员列表更新等，简而言之，每个alertmanager都知道所有的alertmanager的存在
```

### Alertmanager如何基于Gossip实现告警不重复的

Alertmanager使用Gossip协议来进行集群通信。具体的实现流程如下：

1. 首先，Alertmanager节点之间通过Gossip协议建立相互联系。每个Alertmanager节点都会维护一个Gossip池，用于存储其他节点的状态信息。
2. 当集群之中某一个Alertmanager节点接到告警时，首先会计算该告警的Fingerprint（即告警内容的摘要），并将其作为ID使用。
2. 然后Alertmanager将该告警的Fingerprint广播给其他Alertmanager节点。
3. 接收到广播的Alertmanager会将该Fingerprint添加到自己的已知Fingerprint列表中。
4. 当Alertmanager要发送一个告警通知时，会检查该告警的Fingerprint是否已经存在于已知Fingerprint列表中。如果已经存在，则不会发送该告警通知。

也就是说每一个alertmanager节点都拥有所有的告警的Fingerprint，这个Fingerprint列表就是抑制重复发送的ID

### Alertmanager的Gossip协议有哪些特点
```
1. 高效：Gossip协议使用随机化的节点选择和增量式的消息传递方式，能够在短时间内将信息传递给集群中的所有节点。
2. 可靠：Gossip协议采用反馈机制和故障检测算法，能够检测并快速恢复集群中的故障节点，保证集群的稳定性和可靠性。
3. 去中心化：Gossip协议不依赖于任何中心节点或集中式控制器，所有节点都是平等的，能够自组织、自平衡和自适应。
```

### Alertmanager的集群有什么缺点
``` txt
1. 增加了复杂性：Alertmanager集群需要进行配置和管理，增加了系统复杂性。
2. 需要更多的资源：Alertmanager集群需要更多的资源，包括计算、存储和网络等。
3. 需要进行监控和日志管理：Alertmanager集群需要进行监控和日志管理，以便及时发现和解决问题。
5. 信息安全性：Alertmanager集群需要注意安全性，尤其是在跨网络或公共网络上运行时。
```

### alertmanager集群在什么情况下告警仍然重复发送了
``` bash
1. 集群中的某些节点出现网络故障（比如节点1和节点2同时接收到同一个告警，但是节点1和节点2之间无法通讯）
```

### 两台alertmanager同时接收到告警，他们之间通过Gossip通讯，会等待多久才会发送给receiver

``` bash
1. 当某一个Alertmanager实例接收到一个告警，则它会等待一段时间（默认为30秒）观察是否还有其他Alertmanager实例接收到该告警
2. 如果在等待聚合时间结束后，Alertmanager集群仍然没有收到该告警的聚合，则它将发送给接收器（receiver）

# 备注
等待时间称为“等待聚合”（wait_for_aggregate）时间，可以在Alertmanager配置文件的route部分进行配置;
```

![alertmanager接收到告警之后如何处置](/images/am-notifi-pipeline.png)

![prometheus发送给alertmanager后告警流程](/images/am-gossip.png)

### 参考资料

[Alertmanager高可用](https://yunlzheng.gitbook.io/prometheus-book/part-ii-prometheus-jin-jie/readmd/alertmanager-high-availability#chuang-jian-alertmanager-ji-qun)