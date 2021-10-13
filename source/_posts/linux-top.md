---
title: linux top命令
date: 2021-09-18 15:56:48
tags:
- linux
- kubernetes
categories:
- linux
- kubernetes
---

### top命令参数信息

在命令top中可以方便的查看系统的cpu和内存信息(如下图所示), 然后你可能不清楚每一个字段代表的含义，现在就一块揭秘top命令中每个字段的含义

![top command](https://tva1.sinaimg.cn/large/008i3skNly1gut64f02osj61h40octfj02.jpg)

`us`：user time，表示 CPU 执行用户进程的时间，包括 nice 时间。通常都是希望用户空间CPU越高越好。

`sy`：system time，表示 CPU 在内核运行的时间，包括 IRQ 和 softirq。系统 CPU 占用越高，表明系统某部分存在瓶颈。通常这个值越低越好。

`ni`：nice time，具有优先级的用户进程执行时占用的 CPU 利用率百分比。

`id`：idle time，表示系统处于空闲期，等待进程运行。

`wa`：waiting time，表示 CPU 在等待 IO 操作完成所花费的时间。系统不应该花费大量的时间来等待 IO 操作，否则就说明 IO 存在瓶颈。

`hi`：hard IRQ time，表示系统处理硬中断所花费的时间。

`si`：soft IRQ time，表示系统处理软中断所花费的时间。

`st`：steal time，被强制等待（involuntary wait）虚拟 CPU 的时间，此时 Hypervisor 在为另一个虚拟处理器服务

![top cpu command](https://tva1.sinaimg.cn/large/008i3skNly1gut6kntlhpj61hc0u0gqf02.jpg)

#### cpu 平均负载Load Average

​	在上面的top命令中跟cpu相关的信息还有一个叫load average, load average这一列中的三个数值分别表示过去一分钟，五分钟，十五分钟这个节点上的load average。那么到底啥是load average呢?

​	这里的load average表示对CPU资源需求的度量(说的很好，但是等于放屁~)，举个例子你可能就懂了，对于一个单个 CPU 的系统，如果在 1 分钟的时间里，处理器上始终有一个进程在运行，同时操作系统的进程可运行队列中始终都有 9 个进程在等待获取 CPU 资源。那么对于这 1 分钟的时间来说，系统的"load average"就是 1+9=10(这个定义对绝大部分的Unix 系统都适用)。

总结一句话就是: load average 等于单位时间内正在运行的进程+可运行队列的进程(注意，**这里说的是unix系统**)，因此：

- 第一，不论计算机 CPU 是空闲还是满负载，Load Average 都是 Linux 进程调度器中**可运行队列（Running Queue）里的一段时间的平均进程数目。**
- 第二，计算机上的 CPU 还有空闲的情况下，CPU Usage 可以直接反映到"load average"上，什么是 CPU 还有空闲呢？具体来说就是可运行队列中的进程数目小于 CPU个数，这种情况下，单位时间进程 CPU Usage 相加的平均值应该就是"load average"的值。
- 第三，计算机上的 CPU 满负载的情况下，计算机上的 CPU 已经是满负载了，同时还有更多的进程在排队需要 CPU 资源。这时"load average"就不能和 CPU Usage 等同了。比如对于单个 CPU 的系统，CPU Usage 最大只是有 100%，也就 1 个 CPU；而"loadaverage"的值可以远远大于 1，因为"load average"看的是操作系统中可运行队列中进程的个数。

**对于linux系统来说： load average = 可运行队列的进程 + 处于TASK_UNINTERRUPTIBLE状态的进程(D 状态进程)**

> TASK_UNINTERRUPTIBLE 是 Linux 进程状态的一种，是进程为等待某个系统资源而进入了睡眠的状态，并且这种睡眠的状态是不能被信号打断的。俗称D状态进程
>
> 因此，如果发现系统的load average很高，而CPU还是处于空闲状态，说明有很多进程处于阻塞状态，这时候得检查下代码写的是否有问题啦~(通过 ps 命令可以查阅D状态进程的信息)

![load average](https://tva1.sinaimg.cn/large/008i3skNly1gut7nhnp8kj61hc0u0ac602.jpg)

### top命令原理

分析了top命令中cpu字段的含义，你肯定还有疑问top命令中的数值是怎么计算出来的，

### 容器中如何查看cpu信息



### 参考文档

- https://github.com/silenceshell/topic  // 容器中正确展示top信息