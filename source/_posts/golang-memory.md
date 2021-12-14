---
title: golang中的内存分配
date: 2021-11-11 17:21:46
tags:
- golang
categories:
- golang
---

### 概述

Go语言内置运行时(Runtime), 抛弃了传统的内存分配方式，改为自主管理。这样可以自主地实现更好的内存使用模式，比如内存池、预分配等等。这样，不会每次内存分配都需要进行系统调用。

Golang运行时的内存分配算法主要源自 Google 为 C 语言开发的`TCMalloc算法`，全称`Thread-Caching Malloc`。核心思想就是把内存分为多级管理，从而降低锁的粒度。它将可用的堆内存采用二级分配的方式进行管理：每个线程都会自行维护一个独立的内存池，进行内存分配时优先从该内存池中分配，当内存池不足时才会向全局内存池申请，以避免不同线程对全局内存池的频繁竞争。

### TC Malloc原理



### go 内存分配



### make与new的区别



### 参考文档

- [万字长文深入浅出 Golang Runtime](https://zhuanlan.zhihu.com/p/95056679)   // go夜读的分享，推荐去看下go夜读

- [TC Malloc 内存分配原理简析](https://www.cnblogs.com/jiujuan/p/13869547.html)   // 简洁tc malloc的原理

- [图解Go语言内存分配](https://zhuanlan.zhihu.com/p/59125443)   // 绕全成大佬写的，通俗易懂

- [golang内存分配原理及make和new的区别](https://www.cnblogs.com/33debug/p/12068699.html)   // 强烈推荐这篇文档，这位大佬的go系列文章都很牛批

