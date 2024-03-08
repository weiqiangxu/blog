---
hide: true
---


### 1. maven仓库安装

```bash
docker run -d -p 8081:8081 sonatype/nexus3
```

### 2. maven库访问

访问[http://localhost:8081](http://localhost:8081)右上角以管理员登陆，密码在容器之中的文件里面，用户名称是admin登陆


### Q&A

##### 本地安装了nexus，进入了地址http://localhost:8081/，怎么看有哪些包

-  Browse 选项卡中，您可以看到仓库中的所有包


##### Repositories的maven-central和maven-public和maven-release是什么

Maven 默认包含以下三个仓库：

maven-central: 这是 Maven 中央仓库，默认仓库、依赖项经过测试和验证，可以安全使用。
maven-public: 这是 Maven 公共仓库，存储公开的 Maven 依赖项，任何人都可以向 maven-public 仓库上传依赖项，依赖项可能没有经过测试和验证。
maven-release: 这是 Maven 发布仓库，用于存储已发布的 Maven 依赖，maven-release 仓库中的依赖项是稳定可靠的。