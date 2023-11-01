# chartmuseum

```bash
docker run --name=chartmuseum --restart=always -it -d \
  -p 8080:8080 \
  -v ~/charts:/charts \
  -e STORAGE=local \
  -e STORAGE_LOCAL_ROOTDIR=/charts \
  chartmuseum/chartmuseum:v0.12.0
```

```bash
# 在本地测试，如果helm客户端在其他机器，请修改localhost为指定ip
$ helm repo add chartrepo http://localhost:8080
"chartrepo" has been added to your repositories
$ helm repo list
NAME            URL
chartrepo       http://localhost:8080


# 我们创建并打包一个新的chart
$ helm create test
Creating test
$ helm package test
Successfully packaged chart and saved it to: /home/lijinyang/test-0.1.0.tgz
# 将生成的tgz文件放到chartmuseum的文件夹下
$ mv test-0.1.0.tgz ~/charts/

# 然后helm运行helm repo update更新，并搜索
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "chartrepo" chart repository
Update Complete. ⎈ Happy Helming!⎈
$ helm search repo test
NAME            CHART VERSION   APP VERSION     DESCRIPTION
chartrepo/test  0.1.0           1.16.0          A Helm chart for Kubernetes
$ helm show chart chartrepo/test
apiVersion: v2
appVersion: 1.16.0
description: A Helm chart for Kubernetes
name: test
type: application
version: 0.1.0
```

```bash
# 安装helm push 插件
# helm plugin install https://github.com/chartmuseum/helm-push.git

# helm push命令将chart发布到chartmuseum上
# helm push test-0.1.0.tgz chartrepo

# 更新helm repo，搜索刚刚上传的chart。
# helm repo upgrade
# helm search repo chartrepo
NAME                               CHART VERSION  APP VERSION   DESCRIPTION
chartmuseum/test                   0.1.0          1.16.0        A Helm chart for Kubernetes
```