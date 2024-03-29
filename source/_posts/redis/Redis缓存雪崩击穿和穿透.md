---
hide: true
---

# Redis缓存

### 缓存雪崩

> 数据大量过期，大量请求直接访问数据库造成数据库宕机

```
大量数据同时过期
Redis故障宕机
```

### 解决方案
```
双Key策略
后台更新缓存
过期时间均匀设置
```


### 缓存击穿

> 热点数据过期高并发请求冲垮数据库

### 缓存穿透

> 数据库没有数据缓存之中也没有，那么所有的请求都会打到数据库

```
缓存空值或默认值
布隆过滤器快速判断数据是否存在，避免通过查询数据库来判断数据是否存在
```
