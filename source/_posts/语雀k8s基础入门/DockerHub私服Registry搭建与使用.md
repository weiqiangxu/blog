---
hide: true
---
# Docker Hub私服Registry搭建与使用

1. 镜像拉取
```
docker pull registry:2
```

1. 本地创建私服镜像目录
```
mkdir -p /home/docker/repository
```

2. 启动私服和容器(简约版)
```
docker run \
-d \
-p 0.0.0.0:33307:22 \
-p 0.0.0.0:5000:5000 \
registry
```

3. 启动私服和容器(指定磁盘挂载)
```
docker run \
-d -p 0.0.0.0:33307:22 \
-p 0.0.0.0:5000:5000 \
-v /opt/docker-image:/home/docker/repository \
-e SQLALCHEMY_INDEX_DATABASE:sqlite:/opt/docker-image/docker-registry.db \
-e STORAGE_PATH=/opt/docker-image \
registry
```

### 如何使用

1. 查看hub有什么镜像
```
curl http://localhost:5000/v2/_catalog
```

2. 推送镜像
```
#修改daemon.json文件添加私服地址
"insecure-registry": ["dockerhub.kubekey.local","127.0.0.1"]

docker login dockerhub.kubekey.local

```

2. 拉私服的镜像
```
docker pull localhost:5000/images-name:tag
```