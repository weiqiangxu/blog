---
title: 本地启动prometheus服务
index_img: /images/prometheus.jpeg
banner_img: /images/bg/1.jpg
tags:
  - prometheus
categories:
  - prometheus
date: 2023-04-08 06:40:12
---
# 本地存储

``` bash
$ /Users/xuweiqiang/Documents/data
```

``` bash
$ ./prometheus --storage.tsdb.path=/Users/xuweiqiang/Documents/data \
--config.file=/Users/xuweiqiang/Documents/prometheus.yml \
--web.listen-address=:8989
```

