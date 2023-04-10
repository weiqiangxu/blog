---
title: 搭建alertmanager告警服务
index_img: /images/prometheus_icon.jpeg
tags:
  - prometheus
  - 告警
categories:
  - prometheus
date: 2023-04-10 18:40:12
excerpt: 使用docker和自定义指标搭建promethus服务、alertmanager服务，试用邮件告警功能
---

### 启动prometheus

1. 如何配置告警规则
``` yml
# my global config
global:
  scrape_interval: 15s
  evaluation_interval: 15s 
groups:
- name: example
  rules:
  # 配置请求响应时常超过0.5s就告警
  - alert: HighRequestLatency
    expr: job:request_latency_seconds:mean5m{job="myjob"} > 0.5
    for: 10m
    labels:
      severity: page
    annotations:
      summary: High request latency
  # 配置内存低于10%就告警
  - alert: HostOutOfMemory
      expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100 < 10
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: Host out of memory (instance {{ $labels.instance }})
        description: "Node memory is filling up (< 10% left)\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
```

2. 告警方式

```
a. 自己实现定时任务，通过prometheus api接口获取告警数据，然后自己实现发邮件
b. 使用alert manager告警（目前支持邮件、电话吗）
```

### docker启动alert manager
1. 配置
``` bash
$ mkdir -p /Users/xuweiqiang/Documents/alertmanager/
$ mkdir -p /Users/xuweiqiang/Documents/alertmanager/template
$ touch /Users/xuweiqiang/Documents/alertmanager/config.yml
$ touch /Users/xuweiqiang/Documents/alertmanager/template/email.tmpl
$ cd /Users/xuweiqiang/Documents/alertmanager
```
``` bash
$ vim /Users/xuweiqiang/Documents/alertmanager/alertmanager.yml
$ touch /Users/xuweiqiang/Documents/alertmanager/template/email.tmpl
```
``` yml
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
    -v /Users/xuweiqiang/Documents/alertmanager:/etc/alertmanager \
    prom/alertmanager:latest
```

### 启动prometheus
1. 设置
``` bash
$ touch /Users/xuweiqiang/Documents/alertmanager/prometheus.yml
$ vim /Users/xuweiqiang/Documents/alertmanager/prometheus.yml
```
``` yml
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
``` bash
$ mkdir -p /Users/xuweiqiang/Documents/alertmanager/rules
$ vim /Users/xuweiqiang/Documents/alertmanager/rules/one.yml
```
``` yml
groups:
- name: hostStatsAlert
  rules:
  - alert: hostCpuUsageAlert
    expr: (1 - avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance))*100 > 85
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Instance {{ $labels.instance }} CPU usage high"
      description: "{{ $labels.instance }} CPU usage above 85% (current value: {{ $value }})"
  - alert: hostMemUsageAlert
    expr: (1 - (node_memory_MemAvailable_bytes{} / (node_memory_MemTotal_bytes{})))* 100 > 70
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Instance {{ $labels.instance }} MEM usage high"
      description: "{{ $labels.instance }} MEM usage above 70% (current value: {{ $value }})"
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
    -v /Users/xuweiqiang/Documents/alertmanager:/etc/prometheus/ \
    prom/prometheus \
    --config.file=/etc/prometheus/prometheus.yml
```

4. 模拟一个指标就是在本机启动一个request_counter指标

[golang实现简单的指标exporter](https://weiqiangxu.github.io/2023/04/10/prometheus/golang%E5%AE%9E%E7%8E%B0%E7%AE%80%E5%8D%95%E7%9A%84%E6%8C%87%E6%A0%87exporter/)


#### 告警状态

```
Inactive：这里什么都没有发生。
Pending：已触发阈值，但未知足告警持续时间（即rule中的for字段）
Firing：已触发阈值且满足告警持续时间。警报发送到Notification Pipeline，通过处理，发送给接受者这样目的是屡次判断失败才发告警，减小邮件
```

### prometheus告警机制

``` txt
# 简要描述：
Prometheus会根据rules中的规则，不断的评估是否需要发出告警信息,
如果满足规则中的条件，则会向alertmanagers中配置的地址发送告警
告警是通过alertmanager配置的地址post告警,比如targets: [node1:8090']，则会向node1:8090/api/v2/alerts发送告警信息

# 如何验证：
自己实现alertmanger程序，来接收Prometheus发送的告警，并将告警打印出来

# 告警间隔 
[prometheus.yml] > global.evaluation_interval = 15s (告警周期)
[rules.yml] > groups[].rules[].for = 1m (具体某一条告警规则的告警时长)

1m 15s就是prometheus推送告警信息的间隔

# 如何理解
1m 是prometheus拉取指标的间隔 (隔1m拉指标1次)
15s是对应的告警设置的持续时间（需要持续15s以上才会告警）
所以第一次pull到异常时候，到第二次再次pull到异常才满足了15s持续的条件
如果设置为0则立刻告警
```

### alertmanager处理告警信息机制

```
告警分组
抑制
静默
延时
```

### 什么情况下会有Firing状态值



### 参考资料

[Prometheus一条告警是怎么触发的](https://blog.csdn.net/ActionTech/article/details/82421894)
[Prometheus发送告警机制](https://www.cnblogs.com/zydev/p/16848444.html)
[开箱即用的 Prometheus 告警规则集](https://zhuanlan.zhihu.com/p/371967435)
[使用Docker部署alertmanager并配置prometheus告警](https://cloud.tencent.com/developer/article/2211153)