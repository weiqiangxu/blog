---
hide: true
---


1. 使用compose搭建loki服务

``` bash
$ mkdir loki
$ cd loki
$ mkdir data
$ mkdir log
$ mkdir config && cd config && touch local-config.yaml && touch promtail-local-config.yaml
$ touch docker-compose.yml
```

``` yml
# docker-compose.yml
version: '3.7'
networks:
  some-network:
    driver: bridge
services:
  loki:
    networks:
      some-network:
        aliases:
         - alias1
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - ./config:/etc/loki
      - ./data:/data/loki
```

``` yml
# local-config.yaml
# 是否开启认证
auth_enabled: false
# HTTP和gRPC服务监听地址
server:
  http_listen_port: 3100
  grpc_listen_port: 9095
# 配置日志索引和存储
schema_config:
  configs:
    - from: 2018-04-15
      store: boltdb
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

# 配置日志存储后端
storage_config:
  boltdb:
    directory: /data/loki/index
  filesystem:
    directory: /data/loki/chunks

# 配置日志收集
ingester:
  wal:
    enabled: true
    dir: "/tmp/wal"
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
      heartbeat_timeout: 1m
    final_sleep: 0s
  chunk_idle_period: 5m
  chunk_retain_period: 30s
```

```  bash
$ cd loki && sudo docker-compose up -d
```

``` bash
# 目录结构
$ tree .
.
├── config
│   └── local-config.yaml
├── data
│   ├── chunks
│   │   └── loki_cluster_seed.json
│   └── index
└── docker-compose.yml
```

[http://localhost:3100/loki/api/v1/series](http://localhost:3100/loki/api/v1/series)

2. promtail

``` yml
# promtail-config.yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://alias1:3100/loki/api/v1/push

scrape_configs:
- job_name: system
  static_configs:
  - targets:
      - localhost
    labels:
      job: varlogs
      __path__: /var/log/*log

- job_name: file
  static_configs:
  - targets:
      - localhost
    labels:
      job: file
      __path__: /log/audit.log
  pipeline_stages:
    - json:
        expressions:
          name: name
```

``` bash
$ docker run -d \
    --network loki_some-network \
    --network-alias p \
    --name promtail \
    --privileged=true \
    -v /Users/xuweiqiang/Documents/tmp/loki/log:/log \
    -v /Users/xuweiqiang/Documents/tmp/loki/config:/config \
    grafana/promtail:latest -config.file=/config/promtail-config.yaml
```

### 相关资料

[https://hub.docker.com/r/grafana/loki](https://hub.docker.com/r/grafana/loki)
[https://grafana.com/docs/loki/latest/clients/promtail/installation/](https://grafana.com/docs/loki/latest/clients/promtail/installation/)