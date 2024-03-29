---
hide: true
---

# RDB快照

> 某一个瞬间的内存数据

### 命令
```
SAVE
BGSAVE
```

### 优点
```
二进制
数据恢复效率更高
```

### 缺点
```
全量快照开销大
通常5min快照一次，数据丢失量也大
```

### 记录快照和数据修改的冲突
```
写时复制技术（Copy-On-Write, COW）

主进程(写读命令)和bgsave子进程共享所有内存数据

主进程更改数据前是将共享数据复制一份出来修改的(写时复制),极端情况所有内存修改那么所有内存都要复制

谨防内存占满
```

### Redis4.0 混合持久化
```
redis.conf.aof-use-rdb-preamble yes
```
> 准确来说就是既记录内存快照，也记录增量命令到AOF