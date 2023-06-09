---
title: CAP理论
tags:
  - CAP
categories:
  - 分布式
date: 2023-04-08 06:40:12
index_img: /images/bg/computer.jpeg
---

### 分布式和微服务什么意思
```
单机系统就是程序部署到一台机器，所有的服务由这一台机器提供

分布式系统相对而言，是一组为了完成共同任务而协调工作的计算机节点组成,它们通过网络进行通讯的系统；

分布式系统是多个机器共同提供服务
```
```
分布式系统拆分模式

水平切分：水平切分是指将同一个系统部署到多台机器上
垂直切分：垂直切分是按照业务的维度进行拆分，将各个业务独立出来，单独开发和维护
混合切分：混合切分是将水平切分和垂直切分结合起来的一种切分方法


微服务架构 - 大部分都是采用混合切分
```
> 所以说微服务是分布式的一个子集，微服务是将服务拆分，并且分布式部署

### 分布式系统面临的问题

```
1. 分布式计算的八大谬论
2. 通讯异常，网络分区，三态，节点故障
```


### 分布式系统的设计原则

1. CAP原则
2. BASE理论



### CAP

1. Consistency 一致性
```
分布式系统每个副本数据，用户读取数据都是最新的值
```
2. Aviability 可用性
```
客户端向其中一个服务器发起一个请求且该服务器未崩溃，那么这个服务器最终必须响应客户端的请求

反过来说，如果为了强一致性，请求一个服务器的时候，延迟3s等副本同步完成才响应，那么是不满足可用性的
```
3. 分区容错性 Partition tolerance
```
是否容忍消息丢失

信息丢失或者失败不会影响系统的继续运作

比如副本之间同步的消息丢失不影响服务正常提供
```

### 对于一个分布式系统来说，P 是一个基本要求，CAP 三者中，只能根据系统要求在 C 和 A 两者之间做权衡，并且要想尽办法提升 P
### 对于一个分布式系统来说，P 是一个基本要求
### 对于一个分布式系统来说，P 是一个基本要求

### 分布式系统的三种模式

1. AP
```
1.保证服务可用（当某一个副本网络故障不影响）
2.某一个信息丢失不影响系统继续运行

Redis的集群就是这样，多个副本在同一个时刻不保证数据强一致
```

3. CP
```
1. 强一致性（每个时刻每个副本数据一致）
2. 消息丢失也正常提供服务

为什么没办法有Aviability可用性，因为为了满足强一致性，请求服务时候需要阻塞等待副本同步
```

4. AC 
```
1. 可用性
2. 一致性

要任何时刻都立刻响应，且数据多个副本一致，就无法达到 Partition tolerance，因为消息丢失就无法提供服务了（数据一致性不可能）
```

1. Zookeeper 保证的是 CP 也就是强一致性，但是有时候服务会阻塞一下延迟响应
2. Redis集群 是AP，也就是每个时刻都是服务立即可用但是不保证每个副本数据都是最新的


### BASE理论

```
BASE理论中，一致性分为强一致性和弱一致性

强一致性：当用户更新数据之后，必须保证后续线程或者节点都能马上访问到最新的值

弱一致性：当用户更新数据之后，并不能保证后续线程或者节点都能马上访问到最新的值，它只能通过某种方法来保证最后的一致性
```