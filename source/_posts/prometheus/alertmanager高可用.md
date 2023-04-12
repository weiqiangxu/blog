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

### docker搭建alertmanager集群

``` bash
# a1
$ docker run -d \
    --network p_net \
    --network-alias a1 \
    --name=a1 \
    -p 9093:9093 \
    -v /Users/xuweiqiang/Documents/alertmanager:/etc/alertmanager \
    prom/alertmanager:latest \
    --web.listen-address=":9093" \
    --cluster.listen-address="127.0.0.1:8001" \
    --config.file=/etc/prometheus/alertmanager.yml  \
    --storage.path=/data/alertmanager/
```
``` bash
# a2
$ docker run -d \
    --network p_net \
    --network-alias a2 \
    --name=a2 \
    -p 9094:9094 \
    -v /Users/xuweiqiang/Documents/alertmanager:/etc/alertmanager \
    prom/alertmanager:latest \
    --web.listen-address=":9094" \
    --cluster.listen-address="127.0.0.1:8002" \
    --cluster.peer=a1:8001 \
    --config.file=/etc/prometheus/alertmanager.yml  \
    --storage.path=/data/alertmanager2/
```
``` bash
# a3
$ docker run -d \
    --network p_net \
    --network-alias a3 \
    --name=a3 \
    -p 9095:9095 \
    -v /Users/xuweiqiang/Documents/alertmanager:/etc/alertmanager \
    prom/alertmanager:latest \
    --web.listen-address=":9095" \
    --cluster.listen-address="127.0.0.1:8003" \
    --cluster.peer=a1:8001 \
    --config.file=/etc/prometheus/alertmanager.yml  \
    --storage.path=/data/alertmanager3/ \
    --log.level=debug
```
``` bash
# webhook
$ go install github.com/prometheus/alertmanager/examples/webhook
# 启动服务
$ webhook
```
``` yml
# 配置alertmanager向webhook发送告警信息
route:
  receiver: 'default-receiver'
receivers:
  - name: default-receiver
    webhook_configs:
    - url: 'http://127.0.0.1:5001/'
```

### 测试发送告警




### 参考资料

[Alertmanager高可用](https://yunlzheng.gitbook.io/prometheus-book/part-ii-prometheus-jin-jie/readmd/alertmanager-high-availability#chuang-jian-alertmanager-ji-qun)