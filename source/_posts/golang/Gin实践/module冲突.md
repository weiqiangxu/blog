---
title: module冲突
index_img: /images/bg/文章通用.png
tags:
  - golang
categories:
  - golang
date: 2023-03-12 09:40:12
excerpt: 记录一次包版本不兼容导致的冲突和解决办法
---

### 一、问题描述

``` bash
bingokube-nvs/internal/vnetstore imports
        gitlab.bingosoft.net/bingokube/client-go/tools/cache imports
        k8s.io/apimachinery/pkg/util/clock: module k8s.io/apimachinery@latest found (v0.27.3), but does not contain package k8s.io/apimachinery/pkg/util/clock
```

1. `gitlab.bingosoft.net/bingokube/client-go`的 `go.mod` 依赖的版本是`k8s.io/apimachinery v0.22.4`;
2. `gitlab.bingosoft.net/bingokube/client-go/tools/cache` 依赖了 `package k8s.io/apimachinery/pkg/util/clock`;
3. `k8s.io/client-go`依赖了`k8s.io/apimachinery@latest found (v0.27.3)`;
4. `k8s.io/apimachinery@latest found (v0.27.3)`已经移除了包`package k8s.io/apimachinery/pkg/util/clock`;
5. 执行`go mod tidy`之后自动引用了包`k8s.io/apimachinery@latest`作为两个包`k8s.io/client-go`和`gitlab.bingosoft.net/bingokube/client-go`的共同依赖;


### 二、解决包冲突的方式

1. 指定包`apimachinery`版本，看`k8s.io/client-go`和`bingokube/client-go`都兼容

``` bash
# 手动指定版本依赖
$ go mod edit -require k8s.io/apimachinery@v0.22.4
```


``` yml
module new_kube

go 1.20

require (
	gitlab.bingosoft.net/bingokube/client-go v0.22.21
    // 手动指定的版本
	k8s.io/apimachinery v0.22.4
)

// go mod tidy自动整理的依赖
require (
	github.com/davecgh/go-spew v1.1.1 // indirect
	github.com/go-logr/logr v1.2.0 // indirect
	github.com/gogo/protobuf v1.3.2 // indirect
)
```

``` bash
# 查看依赖关系
go mod graph | grep apimachinery
go help mod
``` 

```shell
# 清理已下载的模块缓存
# 该命令会删除 `$GOPATH/pkg/mod/cache` 目录下的所有缓存文件
go clean -modcache

# 清理未使用的模块缓存
# 该命令会检查项目中的依赖，并清理掉没有使用的模块缓存
go mod tidy

# 将 `<module>` 替换为具体的模块路径，该命令会删除指定模块的缓存
# Go mod 的缓存是全局的，清理缓存可能会导致其他项目的构建时间增长
go clean -modcache -i <module>
```

2. 更新`bingokube/client-go`依赖的`apimachinery`版本

就是更改`bingokube/client-go`的代码，让其兼容`apimachinery@latest`;


### 相关文档

[go mod tidy module x found, but does not contain package x](https://budougumi0617.github.io/2019/09/20/fix-go-mod-tidy-does-not-contain-package/)