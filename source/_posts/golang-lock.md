---
title: golang中锁的实现原理
date: 2021-11-22 21:35:17
tags:
- golang
categories:
- golang
---

### 概述

Go 语言作为一个原生支持用户态进程（Goroutine）的语言，当提到并发编程、多线程编程时，往往都离不开锁这一概念。锁是一种并发编程中的同步原语（Synchronization Primitives），它能保证多个 Goroutine 在访问同一片内存时不会出现竞争条件（Race condition）等问题。go语言在Sync包中提供了用于同步的一些基本原语，包括常见的[`sync.Mutex`](https://draveness.me/golang/tree/sync.Mutex)、[`sync.RWMutex`](https://draveness.me/golang/tree/sync.RWMutex)、[`sync.WaitGroup`](https://draveness.me/golang/tree/sync.WaitGroup)、[`sync.Once`](https://draveness.me/golang/tree/sync.Once) 和 [`sync.Cond`](https://draveness.me/golang/tree/sync.Cond)：

![golang-basic-sync-primitives](https://img.draveness.me/2020-01-23-15797104327981-golang-basic-sync-primitives.png)

### 前提知识

##### 悲观锁与乐观锁

悲观锁是一种悲观思想，它总认为最坏的情况可能会出现，它认为数据很可能会被其他人所修改，不管读还是写，悲观锁在执行操作之前都先上锁。

对读对写都需要加锁导致性能低，所以悲观锁用的机会不多。但是在多写的情况下，还是有机会使用悲观锁的，因为乐观锁遇到写不一致的情况下会一直重试，会浪费更多的时间。

乐观锁的思想与悲观锁的思想相反，它总认为资源和数据不会被别人所修改，所以读取不会上锁，但是乐观锁在进行写入操作的时候会判断当前数据是否被修改过。乐观锁的实现方案主要包含CAS和版本号机制。乐观锁适用于多读的场景，可以提高吞吐量。

CAS即Compare And Swap（比较与交换），是一种有名的无锁算法。即不使用锁的情况下实现多线程之间的变量同步，也就是在没有线程被阻塞的情况下实现变量的同步，所以也叫非阻塞同步。CAS涉及三个关系：指向内存一块区域的指针V、旧值A和将要写入的新值B。CAS实现的乐观锁会带来ABA问题，同时整个乐观锁在遇到数据不一致的情况下会触发等待、重试机制，这对性能的影响较大。

版本号机制是通过一个版本号version来实现版本控制。

##### 自旋锁

之前介绍的CAS就是自旋锁的一种。同一时刻只能有一个线程获取到锁，没有获取到锁的线程通常有两种处理方式：

- 一直循环等待判断该资源是否已经释放锁，这种锁叫做自旋锁，它不用将线程阻塞起来(NON-BLOCKING)；
- 把自己阻塞起来，等待重新调度请求，这种是互斥锁。

自旋锁的原理比较简单，如果持有锁的线程能在短时间内释放锁资源，那么那些等待竞争锁的线程就不需要做内核态和用户态之间的切换进入阻塞状态，它们只需要等一等(自旋)，等到持有锁的线程释放锁之后即可获取，这样就避免了用户进程和内核切换的消耗。

但是如果长时间上锁的话，自旋锁会非常耗费性能，它阻止了其他线程的运行和调度。线程持有锁的时间越长，则持有该锁的线程将被OS调度程序中断的风险越大。如果发生中断情况，那么其他线程将保持旋转状态(反复尝试获取锁)，而持有该锁的线程并不打算释放锁，这样导致的是结果是无限期推迟，直到持有锁的线程可以完成并释放它为止。

解决上面这种情况一个很好的方式是给自旋锁设定一个自旋时间，等时间一到立即释放自旋锁。自旋锁的目的是占着CPU资源不进行释放，等到获取锁立即进行处理。

### golang锁的实现

Golang的Mutex其实是在不断改进的，到目前为止Mutex经历的四个阶段的改进，具体可以参考链接二中大佬的博客。

Go 语言的 [`sync.Mutex`](https://draveness.me/golang/tree/sync.Mutex) 由两个字段 `state` 和 `sema` 组成。其中 `state` 表示当前互斥锁的状态，而 `sema` 是用于控制锁状态的信号量。

```go
type Mutex struct {
	state int32
	sema  uint32
}
```

其中state的最低三位mutexLocked, mutexWoken, mutexStarving表示锁的三种状态：

- `mutexLocked` — 表示互斥锁的锁定状态；
- `mutexWoken` — 表示从正常模式被从唤醒；
- `mutexStarving` — 当前的互斥锁进入饥饿状态；
- `waitersCount` — 当前互斥锁上等待的 Goroutine 个数；

![golang-mutex-state](https://img.draveness.me/2020-01-23-15797104328010-golang-mutex-state.png)

互斥锁的加锁过程比较复杂，它涉及自旋、信号量以及调度等概念：

- 如果互斥锁处于初始化状态，会通过置位 `mutexLocked` 加锁；
- 如果互斥锁处于 `mutexLocked` 状态并且在普通模式下工作，会进入自旋，执行 30 次 `PAUSE` 指令消耗 CPU 时间等待锁的释放；
- 如果当前 Goroutine 等待锁的时间超过了 1ms，互斥锁就会切换到饥饿模式；
- 互斥锁在正常情况下会通过 [`runtime.sync_runtime_SemacquireMutex`](https://draveness.me/golang/tree/runtime.sync_runtime_SemacquireMutex) 将尝试获取锁的 Goroutine 切换至休眠状态，等待锁的持有者唤醒；
- 如果当前 Goroutine 是互斥锁上的最后一个等待的协程或者等待的时间小于 1ms，那么它会将互斥锁切换回正常模式；

互斥锁的解锁过程与之相比就比较简单，其代码行数不多、逻辑清晰，也比较容易理解：

- 当互斥锁已经被解锁时，调用 [`sync.Mutex.Unlock`](https://draveness.me/golang/tree/sync.Mutex.Unlock) 会直接抛出异常；
- 当互斥锁处于饥饿模式时，将锁的所有权交给队列中的下一个等待者，等待者会负责设置 `mutexLocked` 标志位；
- 当互斥锁处于普通模式时，如果没有 Goroutine 等待锁的释放或者已经有被唤醒的 Goroutine 获得了锁，会直接返回；在其他情况下会通过 [`sync.runtime_Semrelease`](https://draveness.me/golang/tree/sync.runtime_Semrelease) 唤醒对应的 Goroutine；

### 死锁

​	死锁是指两个或两个以上的进程在执行过程中，由于竞争资源或者由于彼此通信而造成的一种阻塞的现象，若无外力作用，它们都将无法推进下去。此时称系统处于死锁状态或系统产生了死锁，这些永远在互相等待的进程称为死锁进程。

​	例如：两个线程A、B各自持有一个无法共享的资源，并且他们都需要获取对方现在持有的资源才能进行下一步，但是他们又必须等对方释放了才能去获取，于是A等待B，B也在等待A。如此这般，死锁就产生了。

### 产生死锁的条件及预防死锁

死锁的产生必须具备如下四个必要条件：

1. **互斥条件**：资源不能被共享，只能由某一个进程使用
2. **请求和保持**: 已经得到资源的进程可以再次申请新的资源。
3. **不可剥夺条件**: 已经分配的资源不能从相应的进程中被强制地剥夺。
4. **循环等待条件**：系统中若干进程组成环路，该环路中每个进程都在等待相邻进程正占用的资源。

这四个条件太抽象了，现在就以一个例子说明：

​	两个线程各自持有一个无法共享(互斥条件)的资源，并且他们都需要获取（请求与保持条件）对方现在持有的资源才能进行下一步，但是他们又必须等对方释放了才能去获取(不可剥夺条件)，于是A等待B，B也在等待A（环路等待条件）。如此这般，死锁就产生了。

要预防死锁，只需要破坏四个必要条件中的一个或多个，使死锁永远无法满足即可。

### 参考文档

- https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-sync-primitives/#cond  // 面向信仰编程，这文章也挺不错的~
- https://nxw.name/2021/golang-mutexde-shi-xian-yuan-li-1ef30cc7  // 这篇文章值得好好看
- https://cloud.tencent.com/developer/article/1493418   // 死锁的产生条件
- https://segmentfault.com/a/1190000039712353  // 这文章也不错
