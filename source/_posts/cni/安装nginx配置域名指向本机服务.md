---
title: 安装nginx配置域名指向本机服务
index_img: /images/bg/network.png
banner_img: /images/bg/computer.jpeg
tags:
  - kubernetes
categories:
  - kubernetes
date: 2023-09-19 18:40:12
excerpt: docker创建一个Gin服务并安装nginx通过域名指向本机Gin服务
sticky: 1
---

```bash
docker run --name nginx -itd --network=host \
        -v nginx.conf:/etc/nginx/conf.d/proxy.conf \
        nginx:1.25.1
```

```nginx.conf
client_body_buffer_size 2048m;   # 设置客户端请求的缓冲区大小
client_header_buffer_size 2048m; # 设置客户端请求头部的缓冲区大小
client_max_body_size 2048m;      # 设置客户端请求的最大缓冲区大小

server {
    listen 80;
    server_name a.com;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name b.com;

    location / {
        proxy_pass http://127.0.0.1:5100;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 启动服务指向5100端口

```bash
docker run -itd --name chartmuseum \
        --network=host \
        -e STORAGE=local \
        -e STORAGE_LOCAL_ROOTDIR=/charts \
        -e PORT=5100 \
        chartmuseum:1.0
```

### 使用IP访问

http://b.com