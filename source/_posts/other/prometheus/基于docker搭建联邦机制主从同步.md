---
title: prometheus的配置重载方案
---

官方提供了 SIGHUP通过向 Prometheus 进程发送 a 或向端点发送 HTTP POST 请求/-/reload 的方式重载配置；
发送 -/reload 的时候，触发 <-chan os.Signal，调用reloadConfig函数更新 reloader(config.Config)
reloadConfig函数仍然是基于yml_file_content解析
所以我打算将 yml_file_content 存储到 etcd 并且watch这个key的变化


1. main.main函数启动时候更改 config.LoadFile(cfg.configFile 为 config.LoadConfigFromEtcd(cfg.configFile,
2. 在 <-hub (chan os.Signal) 监听的select之中添加 <-etcd.Listen() 监听，有配置更改时候调用 reladConfig 函数


### prometheus的多台写方案

1. protheus pull 数据的时候加入拦截器，在API层获取到拉取到的数据包，甩给其他的prometheus客户端的API
2. 本地队列 + 异步队列
3. 如果写失败了如何保证消息不丢失


### 当前存在的问题

单台机器pull并存储如果节点故障就会无法使用

扩展多个p的实例，多个p对metrics进行pull并写入，互不干扰，但是对于客户端又多了很多重复的网络io占用

所以决定让多个p的实例之中的其中一个pull数据，通过etcd实现的锁，控制同一时刻只有一个p的客户端pull数据

在p拉取数据的时候，在API层实现拦截器，拉取到数据时候调用其他p的api接口写数据，代码侵入最小也是最简单可靠的方案

但是写数据是异步+本地队列的方式：    

1. 其他P宕机、写数据失败等情况出现多个P的数据一致性问题
2. 单机的存储有限，数据量的问题 - 这里暂时不考虑按业务分片 - 后续要考虑数据迁移的问题


[快猫监控P高可用](http://flashcat.cloud/docs/content/flashcat-monitor/prometheus/ha/local-storage/)

[本地存储配置](https://blog.csdn.net/m0_60244783/article/details/127641195)

### prometheus exporter 有哪些？

### prometheus在zookeeper数据变化时候直接更改拉取规则不重启如何实现?



### 使用 federation 实现数据一致性

https://prometheus.io/docs/prometheus/latest/federation/

1. docker install两个prometheus
2. 本地mac启动一个exporter暴露系统指标
3. 指定一个prometheus采集指标
4. federation机制让另一个prometheus也采集到一样的指标


### mac的本机器指标
```
https://prometheus.io/download/

./node_exporter

http://localhost:9100/metrics
```

### 启动docker并采集本机器指标
```
docker exec -it --user root ${容器id} /bin/sh
```
```
docker network create p_net

docker run \
    --name master \
    -d \
    -p 9090:9090 \
    --network p_net \
    --network-alias master \
    -v /Users/xuweiqiang/Documents/code/book/other/prometheus/master.yml:/etc/prometheus/prometheus.yml \
    prom/prometheus \
    --query.lookback-delta=15d \
    --config.file=/etc/prometheus/prometheus.yml
```
```
./prometheus --query.lookback-delta=15d --config.file=/prometheus/config.yml
```
[http://localhost:9090/](http://localhost:9090/)

```
# master.yml
# master
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "request_count"
    metrics_path: '/metrics'
    static_configs:
      - targets: ["docker.for.mac.host.internal:6969"] # 宿主机IP ifconfig获取 en0 的IP
```

### 安装另一个prometheus

```
docker run \
    --name slave \
    -d \
    -p 8989:9090 \
    --network p_net \
    --network-alias slave \
    -v /Users/xuweiqiang/Documents/code/book/other/prometheus/slave.yml:/etc/prometheus/prometheus.yml \
    prom/prometheus
```

```
# slave.yml
global:
  scrape_interval: 15s 
  evaluation_interval: 15s 

scrape_configs:
  - job_name: 'federate'
    scrape_timeout: 15s # timeout limit small than scrape_interval
    body_size_limit: 0 # no limit size
    scrape_interval: 15s
    honor_labels: true # 保留原有metrics的标签
    metrics_path: '/federate'
    params:
      'match[]':
        - '{__name__=~".+"}'
    static_configs:
      - targets:
        - 'master:9090'
    # Endpoint的标签
    # relabel_configs:
    #  - target_label: 'instance'
    #    replacement: 'docker.for.mac.host.internal:6969'
```

[http://localhost:8989/](http://localhost:8989/)


### docker exec容器报错su: must be suid to work properly
```
docker exec -ti --user root 容器id /bin/sh
```

### 在容器中访问宿主机服务
```
ifconfig docker0 网卡IP

daemon.json 中定义的虚拟网桥来与宿主机进行通讯 docker.for.mac.host.internal 域名
```

[https://www.ifsvc.cn/posts/156](https://www.ifsvc.cn/posts/156)



### 配置多台prometheus，当选主之后，主节点正常读取数据，备节点通过/federate端点从主节点拉取数据
```
1.prometheus的主节点和副节点的配置不一样么
```

### prometheus federate拉取所有指标 chatGPT
```
# slave
global:
  scrape_interval: 15s 
  evaluation_interval: 15s 

scrape_configs:
  - job_name: 'federate'
    scrape_interval: 15s
    honor_labels: true # 保留原有metrics的标签
    metrics_path: '/federate'
    params:
      'match[]':
        - '{__name__=~".+"}'
    static_configs:
      - targets:
        - 'master:9090'
    # Endpoint的标签
    relabel_configs:
     - target_label: 'instance'
       replacement: 'docker.for.mac.host.internal:6969'
```

### 健康检查接口

[http://localhost:8989/-/healthy](http://localhost:8989/-/healthy)

### 基于ETCD + 3台prometheus 实现高可用 - 主节点保活

1. 主节点配置 scrape_configs 直接从exporter_node拉取数据
2. 从节点配置 scrape_configs 从主节点通过 federate机制同步数据
3. 每台prometheus守护进程中有一个定时器从 etcd 获取主节点的IP，通过/-/health判定主节点的存活状态
4. 如果主节点挂了，选主，将新的主IP同步至etcd，并且更改各个节点的 prometheus配置
5. 如果主节点挂了，发送告警
6. 主节点拉取数据，从节点继续从主节点同步数据

### web站点的配置更改之后，prometheus配置热加载如何实现

### 基于ETCD的集群选主设计方案

1. master节点直接从http接口拉取数据
2. node节点从master/federate端口拉取数据
3. master节点存活信息存储在etcd(etcd有一个TTL key)，master节点每隔30s发送一次心跳，重新设置TTL key否则任务master节点已经挂了
4. master节点挂了以后，剩下的节点竞选 - master节点出来以后，更新master节点的配置和更新node节点的配置，主要是实现主从

### 疑问：

1. 多个节点的保活机制，实现
2. 多个节点的选主算法实现
3. test.yml 文件配置如何写入，我可以直接把二进制内容存储到etcd吗还是转成json写入呢

