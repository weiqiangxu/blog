---
title: 基于ETCD的配置热加载
---

### 启动一个master节点服务

```bash
docker network create etcd_net

docker run \
    --name etcd_master \
    -d \
    -p 7979:9090 \
    --network etcd_net \
    --network-alias master \
    -v /Users/xuweiqiang/Documents/code/book/other/prometheus/etcd_reload_config.yml:/etc/prometheus/prometheus.yml \
    prom/prometheus
```
[localhost:7979](http://localhost:7979)

### 在etcd设置一个target metrics

```
etcdctl set /services/prometheus/example '{"targets":["docker.for.mac.host.internal:6969"],"metrics_path":"/metrics"}'
```

### etcd reload配置

```
# etcd reload config prometheus.yml
global:
  scrape_interval: 5s
  evaluation_interval: 5s
# scrape_configs:
#   - job_name: 'etcd'
#     service_discovery_configs:
#       - etcd_sd_configs:
#           - scheme: "http"
#             endpoints:
#               - "http://docker.for.mac.host.internal:2379"
#             path: "/services/prometheus"
#   - job_name: my-app
#     etcd_sd_configs:
#     - endpoints:
#         - http://127.0.0.1:2379
#       role: endpoints
#       prefix: /my-app
etcd_sd_configs:
  - api_server: 'http://docker.for.mac.host.internal:2379'
    prefix: '/prometheus'
    refresh_interval: 5s
    target_group: 'etcd'
    label_selector: 'job=etcd'
```