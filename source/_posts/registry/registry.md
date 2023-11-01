# registry

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

