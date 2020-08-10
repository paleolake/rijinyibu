---
layout: default
title: 第1章　简介
---

## 第1章　简介

### 1.3.3 性能问题

>在多线程程序中，当线程调度器临时挂起活跃线程并转而运行另一个线程时，就会频繁地出现上下文切换操作（ContextSwitch），这种操作将带来极大的开销：保存和恢复执行上下文，丢失局部性，并且CPU时间将更多地花在线程调度而不是线程运行上。当线程共享数据时，必须使用同步机制，而这些机制往往会抑制某些编译器优化，使内存缓存区中的数据无效，以及增加共享内存总线的同步流量。

~上下文切换操作会丢失局部性，为什么会这样呢？这是因为当切换去运行另一个线程时，那么缓存中的指令和数据都有可能会失效。