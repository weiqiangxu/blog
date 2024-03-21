---
title: 配置Nginx服务SSL
index_img: /images/bg/network.png
banner_img: /images/bg/computer.jpeg
tags:
  - docker
categories:
  - docker
date: 2024-01-19 18:40:12
excerpt: 如何生成SSL证书并且应用于Nginx服务开启https
sticky: 1
---


1. 创建证书

```bash
mkdir cert
```

2. 生成证书

```bash
# 下载openssl并安装配置环境变量
$ openssl version 
LibreSSL 3.3.6


# domain.key 保存私钥的文件
$ openssl genrsa -out domain.key 2048

# 保存CSR请求的文件，会提示输入Country\Province等全部随便输入即可
$ openssl req -new -key domain.key -out domain.csr

$ ls
domain.csr	domain.key

# 将CSR文件提交给您选择的CA。CA会审核您的信息，并颁发SSL证书

# 获取SSL证书文件
```

3. 配置给Nginx

```bash
docker network create nginx-test

docker run --network nginx-test -itd \
  --network-alias gin \
  435861851/gin:v0.0.1
```

```bash
# nginx.conf
server {
    listen 443 ssl;
    server_name b.com;
    ssl_certificate /home/cert/domain.crt;
    ssl_certificate_key /home/cert/domain.key;
}

client_body_buffer_size 2048m;   # 设置客户端请求的缓冲区大小
client_header_buffer_size 2048m; # 设置客户端请求头部的缓冲区大小
client_max_body_size 2048m;      # 设置客户端请求的最大缓冲区大小

server {
    listen 80;
    server_name b.com;

    location / {
        proxy_pass http://gin:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
docker run --name nginx \
  --network nginx-test -itd \
  -p 80:80 \
  -v ./nginx.conf:/etc/nginx/conf.d/proxy.conf \
  nginx:1.25.1
```