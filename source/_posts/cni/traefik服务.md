# traefik


### 一、docker配置traefik访问Nginx服务

```yml
version: '3'

services:
  reverse-proxy:
    # The official v2 Traefik docker image
    image: traefik:v2.10
    # Enables the web UI and tells Traefik to listen to docker
    command: --api.insecure=true --providers.docker
    ports:
      # The HTTP port
      - "80:80"
      # The Web UI (enabled by --api.insecure=true)
      - "8080:8080"
    volumes:
      # So that Traefik can listen to the Docker events
      - /var/run/docker.sock:/var/run/docker.sock
  nginx:
   image: nginx
   labels:
     - "traefik.http.routers.nginx.rule=Host(`yourdomain.com`)"
     - "traefik.http.services.nginx.loadbalancer.server.port=80"
   restart: always
```
```bash
$ docker-compose up -d nginx

$ docker-compose up -d reverse-proxy
```

```bash
# 127.0.0.1 yourdomain.com
$ vim /etc/hosts
```

- [访问Nginx服务yourdomain.com](yourdomain.com)

- [访问traefik可视化界面](http://127.0.0.1:8080/dashboard/#/)


### 二、kubernetes配置traefik访问Nginx服务

要在Kubernetes中配置Traefik访问Nginx服务，你需要执行以下步骤：

1. 首先，确保已经在Kubernetes集群中部署了Traefik Ingress Controller。可以使用以下命令来部署Traefik：

```bash
$ kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v2.5/examples/k8s/traefik-deployment.yaml
```

2. 然后，创建一个Nginx服务的Deployment和Service。可以使用以下示例来创建Nginx Deployment和Service：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
 name: nginx-deployment
spec:
 replicas: 1
 selector:
   matchLabels:
     app: nginx
 template:
   metadata:
     labels:
       app: nginx
   spec:
     containers:
     - name: nginx
       image: nginx:latest
       ports:
       - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
 name: nginx-service
spec:
 selector:
   app: nginx
 ports:
   - protocol: TCP
     port: 80
```

保存上述内容为`nginx.yaml`文件，然后使用以下命令创建Nginx Deployment和Service：

```bash
$ kubectl apply -f nginx.yaml
```

3. 接下来，创建一个Traefik Ingress资源来将Traefik与Nginx服务关联起来。可以使用以下示例来创建Traefik Ingress资源：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
 name: nginx-ingress
 annotations:
   kubernetes.io/ingress.class: traefik
spec:
 rules:
 - host: example.com  # 将example.com替换为你的域名
   http:
     paths:
     - path: /
       pathType: Prefix
       backend:
         service:
           name: nginx-service
           port:
             number: 80
```

保存上述内容为`traefik-ingress.yaml`文件，然后使用以下命令创建Traefik Ingress资源：

```bash
$ kubectl apply -f traefik-ingress.yaml
```

4. 等待一段时间，Traefik Ingress Controller将自动检测到新的Ingress资源，并将其配置到Traefik代理中。然后，你可以使用域名（例如`example.com`）访问Nginx服务。

请确保将`example.com`替换为你的实际域名，并根据实际需求进行其他配置更改，例如TLS证书等。

希望以上步骤能帮助到你配置Traefik访问Nginx服务。