---
title: registry
index_img: /images/bg/k8s.webp
banner_img: /images/bg/5.jpg
tags:
  - registry
categories:
  - kubernetes
date: 2023-07-05 18:40:12
excerpt: 了解registry仓库的安装和使用
sticky: 1
hiden: true
---

```bash
docker run -d -p 5005:5000 --restart=always --name registry registry:latest
```


[http://localhost:5005/v2/_catalog](http://localhost:5005/v2/_catalog)

```bash
docker pull nginx:alpine

docker tag nginx:alpine 127.0.0.1:5005/test/mynginx:v1
```

```bash
curl http://localhost:5005/v2/_catalog

{
  "repositories": [
    "test/mynginx"
  ]
}
```

```bash
# 其他节点拉取镜像
docker pull x.x.x.x:5005/test/mynginx:v1
```

