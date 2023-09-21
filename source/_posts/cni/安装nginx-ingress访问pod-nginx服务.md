---
title: 安装nginx-ingress访问pod-nginx服务
index_img: /images/bg/network.png
banner_img: /images/bg/computer.jpeg
tags:
  - kubernetes
categories:
  - kubernetes
date: 2023-09-19 18:40:12
excerpt: 创建nginx-pod和相关service
sticky: 1
---

### 1. nginx pod

```yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-ingress
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

### 2. nginx service

```yml
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress-service
spec:
  selector:
    app.kubernetes.io/name: nginx-ingress
  ports:
    - protocol: TCP
      port: 8989
      targetPort: 80
```

### 3. ingress-nginx 

[install-ingress-nginx.yaml](https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml)

```bash
$ helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace


# 更改镜像地址
# image: k8s.gcr.io/ingress-nginx/controller:v1.1.1@sha256:0bc88eb15f9e7f84e8e56c14fa5735aaa488b840983f87bd79b1054190e660de
# 变成
# image: registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller:v1.1.1

# image: k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1@sha256:64d8c73dca984af206adf9d6d7e46aa550362b1d7a01f3a0a91b20cc67868660
# 变成
# image: registry.cn-hangzhou.aliyuncs.com/google_containers/kube-webhook-certgen:v1.1.1

$ kubectl apply -f install-ingress-nginx.yaml
```

### 4. 创建ingress yaml

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx-example
  rules:
  - http:
      paths:
      - path: /testpath
        pathType: Prefix
        backend:
          service:
            name: nginx-ingress-service
            port:
              number: 8989
```

``` bash
$ kubectl get service -n ingress-nginx
```