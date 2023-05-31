---
title: 在k8s之中部署loki服务存储日志
index_img: /images/bg/k8s.webp
banner_img: /images/bg/5.jpg
tags:
  - kubernetes
categories:
  - kubernetes
date: 2023-04-23 18:40:12
excerpt: 搭建loki服务存储日志
sticky: 1
---

1. 安装helm

在k8s集群中安装helm，可以使用以下命令：

[https://github.com/helm/helm/releases](https://github.com/helm/helm/releases)

只需要二进制程序下载后移动到 `/usr/local/bin` 目录。

2. 添加loki和promtail helm仓库

使用以下命令添加loki和promtail的helm仓库：

``` bash
$ helm repo add grafana https://grafana.github.io/helm-charts
```

3. 创建命名空间

``` bash
$ kubectl create namespace loki
```

4. 安装loki

使用以下命令安装loki

``` bash
$ helm install loki grafana/loki-stack --namespace=loki
```

5. 安装promtail

使用以下命令安装promtail

``` bash
$ helm install promtail grafana/promtail --namespace=loki
```

6. 配置promtail

在promtail的values.yaml文件中配置promtail的日志采集目标。

``` yml
# /etc/promtail/config.yaml
promtail:
 config:
   server:
     http_listen_port: 9080
     grpc_listen_port: 0
   positions:
     filename: /tmp/positions.yaml
   clients:
     - url: http://loki:3100/loki/api/v1/push
   scrape_configs:
     - job_name: kubernetes-logs
       kubernetes_sd_configs:
         - role: pod
       relabel_configs:
         - source_labels: [__meta_kubernetes_pod_container_name]
           action: replace
           target_label: container_name
         - source_labels: [__meta_kubernetes_pod_name]
           action: replace
           target_label: job
         - source_labels: [__meta_kubernetes_namespace]
           action: replace
           target_label: namespace
```

7. 部署promtail

使用以下命令部署promtail

``` bash
$ helm upgrade --install promtail grafana/promtail --namespace=loki -f config.yaml
```

至此，loki和promtail均已部署完成。可以通过以下命令查看loki和promtail的状态：

``` bash
$ kubectl get pods --namespace=loki
```


8. 创建service直接外部访问loki的api接口

``` bash
# 查看loki的详细的label
$ kubectl describe pod loki-0 -n loki
```

``` yml
apiVersion: v1
kind: Service
metadata:
  name: loki-service
spec:
  type: NodePort
  selector:
    app: loki
  ports:
    - port: 80
      targetPort: 3100
      nodePort: 30007
```

``` yml
apiVersion: v1
kind: Service
metadata:
  name: loki-service
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: loki
  ports:
  - name: loki-port
    protocol: TCP
    port: 80
    targetPort: 3100
    nodePort: 30009
```

``` bash
$ kubectl exec -it loki-0  -n loki -- /bin/sh
$ kubectl apply -f loki-service.yml
$ cd /etc/loki/
```