---
title: alertmanager的历史告警
index_img: /images/prometheus_icon.jpeg
banner_img: /images/bg/5.jpg
tags:
  - alertmanager
  - 告警
  - 高可用
categories:
  - prometheus
date: 2022-07-28 10:13:01
---


### 一、历史告警存档方案 

[官方webhook集成的方法](https://prometheus.io/docs/operating/integrations/#alertmanager-webhook-receiver)

1. alertsnitch + MySQL
2. alertmanager-webhook-logger

### 二、Alertmanager的本地存储

1. 本地存储格式和查看方式

``` bash
Alertmanager可以通过本地存储存档文件格式为JSON。可以通过以下方法查看历史告警：

1. 通过Alertmanager的Web界面查看历史告警

在Alertmanager的Web界面中，可以通过点击"Alerts"按钮查看当前所有的告警信息

2. 通过API接口查看历史告警

Alertmanager提供了多种API接口，可以通过curl等工具来查询历史告警信息。
/api/v1/alerts接口可以用于查询当前活动的告警
而/api/v1/alerts/history接口用于查询历史告警信息。

例如，可以通过以下命令来查询最近一小时内的所有告警：

curl -XGET 'http://localhost:9093/api/v1/alerts/history?start='$(date -d '-1 hours' '+%s')'&end='$(date '+%s')

以上命令中，-XGET用于指定HTTP请求方法为GET，start和end参数用于指定查询时间范围。

3. 通过本地存储文件查看历史告警

Alertmanager的本地存储文件默认存储在/var/lib/prometheus/alertmanager/目录下，直接打开文件
```

### 服务搭建

#### a.启动prometheus

1. prometheus配置

``` bash
$ mkdir -p /Users/xuweiqiang/Desktop/alert
$ mkdir -p /Users/xuweiqiang/Desktop/alert/rules
$ touch /Users/xuweiqiang/Desktop/alert/prometheus.yml
$ touch /Users/xuweiqiang/Desktop/alert/rules/one.yml
```
``` yml
# prometheus.yml
global:
  scrape_interval:     5s 
  evaluation_interval: 5s 

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - alert:9093

scrape_configs:
  - job_name: "request_count"
    metrics_path: '/metrics'
    static_configs:
      - targets: ["docker.for.mac.host.internal:6969"] # 宿主机IP ifconfig获取 en0 的IP
# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  - "/etc/prometheus/rules/*.yml"
```

``` yml
# one.yml
groups:
- name: hostStatsAlert
  rules:
  - alert: request_counter
    expr: app_system_request > 3
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Instance {{ $labels.instance }} request count too much"
      description: "{{ $labels.instance }} request count above 3 (current value: {{ $value }})"
```

2. 启动
``` bash
$ docker run \
    --name alert_client \
    -d \
    -p 9090:9090 \
    --network p_net \
    --network-alias alert_client \
    -v /Users/xuweiqiang/Desktop/alert:/etc/prometheus/ \
    prom/prometheus \
    --config.file=/etc/prometheus/prometheus.yml
```

4. 模拟一个指标就是在本机启动一个request_counter指标

[golang实现简单的指标exporter](https://weiqiangxu.github.io/2023/04/10/prometheus/golang%E5%AE%9E%E7%8E%B0%E7%AE%80%E5%8D%95%E7%9A%84%E6%8C%87%E6%A0%87exporter/)

#### b.启动alertmanager

1. 创建配置

``` bash
$ mkdir -p /Users/xuweiqiang/Desktop/manager
$ mkdir -p /Users/xuweiqiang/Desktop/manager/data
$ mkdir -p /Users/xuweiqiang/Desktop/manager/template
$ touch /Users/xuweiqiang/Desktop/manager/alertmanager.yml
$ touch /Users/xuweiqiang/Desktop/manager/template/email.tmpl
```

``` yml
# alertmanager.yml
global:
  resolve_timeout: 5m
  smtp_from: '435861851@qq.com' # 发件人
  smtp_smarthost: 'smtp.qq.com:587' # 邮箱服务器的 POP3/SMTP 主机配置 smtp.qq.com 端口为 465 或 587
  smtp_auth_username: '435861851@qq.com' # 用户名
  smtp_auth_password: '123' # 授权码 
  smtp_require_tls: true
  smtp_hello: 'qq.com'
templates:
  - '/etc/alertmanager/template/*.tmpl'
route:
  group_by: ['alertname'] # 告警分组
  group_wait: 5s # 在组内等待所配置的时间，如果同组内，5 秒内出现相同报警，在一个组内出现。
  group_interval: 5m # 如果组内内容不变化，合并为一条警报信息，5 分钟后发送。
  repeat_interval: 5m # 发送告警间隔时间 s/m/h，如果指定时间内没有修复，则重新发送告警
  receiver: 'email' # 优先使用 wechat 发送
  routes: #子路由，使用 email 发送
  - receiver: email
    match_re:
      serverity: email
receivers:
- name: 'email'
  email_configs:
  - to: '435861851@qq.com' # 如果想发送多个人就以 ',' 做分割
    send_resolved: true
    html: '{{ template "email.html" . }}'   #使用自定义的模板发送
```

``` html
<!-- email.tmpl -->
{{ define "email.html" }}
{{ range $i, $alert :=.Alerts }}
========监控报警==========<br>
告警状态：{{   .Status }}<br>
告警级别：{{ $alert.Labels.severity }}<br>
告警类型：{{ $alert.Labels.alertname }}<br>
告警应用：{{ $alert.Annotations.summary }}<br>
告警主机：{{ $alert.Labels.instance }}<br>
告警详情：{{ $alert.Annotations.description }}<br>
触发阀值：{{ $alert.Annotations.value }}<br>
告警时间：{{ $alert.StartsAt.Format "2006-01-02 15:04:05" }}<br>
========end=============<br>
{{ end }}
{{ end }}
```

2. 启动

``` bash
$ docker run -d \
    --network p_net \
    --network-alias alert \
    --name=alertmanager \
    -p 9093:9093 \
    -v /Users/xuweiqiang/Desktop/manager:/etc/alertmanager \
    prom/alertmanager:latest \
    --config.file=/etc/alertmanager/alertmanager.yml
```

### 参考资料

[Alertmanager告警全方位讲解](https://blog.csdn.net/agonie201218/article/details/126243110)
[基于Alertmanager设计告警降噪系统-转转](https://zhuanlan.zhihu.com/p/598739724)
[webhook-receiver.go](https://pshizhsysu.gitbook.io/prometheus/ff08-san-ff09-prometheus-gao-jing-chu-li/kuo-zhan-yue-du/shi-jian-ff1a-alertmanager#fu-lu)
[官方webhook reveiver集成写法](https://prometheus.io/docs/operating/integrations/#alertmanager-webhook-receiver)
[查看webhook标准写法](https://github.com/tomtom-international/alertmanager-webhook-logger)
[Alertsnitch: saves alerts to a MySQL database](https://gitlab.com/yakshaving.art/alertsnitch)

