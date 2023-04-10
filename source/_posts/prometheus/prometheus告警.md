---
title: prometheus告警
index_img: /images/prometheus.jpeg
tags:
  - prometheus
  - 告警
categories:
  - prometheus
date: 2023-04-10 06:40:12
excerpt: 如何配置常用的告警规则以及自定义告警推送
---

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

2. 如何通过api接口获取告警推送

``` bash
$ curl http://localhost:9090/api/v1/alerts

{
    "data": {
        "alerts": [
            {
                "activeAt": "2018-07-04T20:27:12.60602144+02:00",
                "annotations": {},
                "labels": {
                    "alertname": "my-alert"
                },
                "state": "firing",
                "value": "1e+00"
            }
        ]
    },
    "status": "success"
}
```

### 参考资料

[开箱即用的 Prometheus 告警规则集](https://zhuanlan.zhihu.com/p/371967435)
[官方手册prometheus如何配置告警规则](https://prometheus.io/docs/alerting/latest/overview/)