---
title: 协程让出抢占调度
---

### 关键术语
1. time.sleep后 _Grunning 和 _Gwaiting，timer之中的回调函数将g变成Grunnable状态放回runq
2. 以上谁负责执行timer之中的回调函数呢 (schedule()->checkTimers)
3. 监控线程（重复执行某一个任务） - 不依赖GMP、main.goroutine创建 ， 监控timer可以创建线程执行
4. IO时间监听队列 - 主动轮询netpoll