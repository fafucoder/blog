---
title: linux cgroup资源限制
date: 2021-08-24 15:05:13
tags:
- linux
categories:
- linux
---

### Cgroup简述

​		在Docker中，容器使用Linux namespace技术进行资源隔离，使得容器中的进程看不到别的容器的资源，但是容器内的进程仍然可以任意地使用主机的 CPU 内存等资源，如果某一个容器使用的主机资源过多，可能导致主机的资源竞争，进而影响业务。那如果我们想限制一个容器资源的使用（如 CPU、内存等）应该如何做呢？

​		这里就需要用到 Linux 内核的另一个核心技术cgroups。那么究竟什么是cgroups？cgroups（全称：Control Groups）是 Linux 内核的一个功能，它可以实现限制进程或者进程组的资源（如 CPU、内存、磁盘 IO 等）。

> 在 2006 年，Google 的工程师（ Rohit Seth 和 Paul Menage 为主要发起人） 发起了这个项目，起初项目名称并不是cgroups，而被称为进程容器（process containers）。在 2007 年cgroups代码计划合入Linux 内核，但是当时在 Linux 内核中，容器（container）这个词被广泛使用，并且拥有不同的含义。为了避免命名混乱和歧义，进程容器被重名为cgroups，并在 2008 年成功合入 Linux 2.6.24 版本中。cgroups目前已经成为 systemd、Docker、Linux Containers（LXC） 等技术的基础。

### Cgroup 功能组件

cgroups功能的实现依赖于三个核心概念：子系统、控制组、层级树。

- **控制组（cgroup）**： 控制组是对进程分组管理的一种机制,一个Cgroup包含一组进程,并可以在上面添加添加Linux Subsystem的各种参数配置,将一组进程和一组Subsystem的系统参数关联起来。
- **子系统（subsystem）**：是一个内核的组件，一个子系统代表一类资源调度控制器。例如内存子系统可以限制内存的使用量，CPU 子系统可以限制 CPU 的使用时间。不同版本的Kernel支持的子系统有所偏差,可以通过`cat /proc/cgroups `查看。
- **层级树（hierarchy）**：是由一系列的控制组(Cgroup) 按照树状结构排列组成的。这种排列方式可以使得控制组(Cgroup) 拥有父子关系，子控制组默认拥有父控制组的属性，也就是子控制组会继承于父控制组。比如，系统中定义了一个控制组 c1，限制了 CPU 可以使用 1 核，然后另外一个控制组 c2 想实现既限制 CPU 使用 1 核，同时限制内存使用 2G，那么 c2 就可以直接继承 c1，无须重复定义 CPU 限制。每个 hierarchy` 在初始化时会有默认的 `CGroup`(`Root CGroup)。
- **任务（task）**:  进程(`process`)在cgroups中称为task，`taskid`就是`pid`。
- **libcgroups**：一个开源软件，提供了一组支持cgroups的应用程序和库，方便用户配置和使用cgroups。目前许多发行版都附带这个软件。

![cgroup组件](https://tva1.sinaimg.cn/large/008i3skNly1gts1dsuihnj60yc0f2dgn02.jpg)

#### linux 支持的子系统

不同版本的Kernel支持的子系统有所偏差,可以通过`cat /proc/cgroups `查看。

- `blkio` 对块设备(比如硬盘)的IO进行访问限制
- `cpu` 设置进程的`CPU`调度的策略, 比如`CPU`时间片的分配
- `cpuacct` 统计/生成`cgroup`中的任务占用CPU资源报告
- `cpuset` 在多核机器上分配给任务(`task`)独立的`CPU`和内存节点(内存仅使用于`NUMA`架构)
- `devices` 控制`cgroup`中对设备的访问
- `freezer` 挂起(suspend) / 恢复 (resume)`cgroup` 中的进程
- `memory` 用于控制`cgroup`中进程的占用以及生成内存占用报告
- `net_cls` 使用等级识别符（classid）标记网络数据包，这让 Linux 流量控制器 **tc** (`traffic controller`) 可以识别来自特定 cgroup 的包并做限流或监控
- `net_prio` 设置`cgroup`中进程产生的网络流量的优先级
- `hugetlb` 限制使用的内存页数量
- `pids` 限制任务的数量
- `ns` 可以使不同`cgroups`下面的进程使用不同的`namespace`. 每个`subsystem`会关联到定义的`cgroup`上,并对这个`cgoup`中的进程做相应的限制和控制.

![cgroups](https://tva1.sinaimg.cn/large/008i3skNly1gts4tgglopj61au0hotai02.jpg)

### 挂载cgroup 文件系统

```shell
# 创建目录
mkdir mycgroup

# 挂载cgroup
mount -t cgroup -o none,name=mycgroup mycgroup `pwd`/mycgroup

# 查看是否挂载成功
mount -t cgroup | grep mycgroup

# 创建子目录
mkdir -p mycgroup/cgroup-1 && mkdir -p mycgroup/cgroup-2
```

执行完上面的命令后查看mycgroup目录发现多了几个文件:

```shell
[root@node1 mycgroup]# ls
cgroup-1  cgroup-2  cgroup.clone_children  cgroup.procs  cgroup.sane_behavior  notify_on_release  release_agent  tasks

[root@node1 mycgroup]# tree .
.
├── cgroup-1
│   ├── cgroup.clone_children
│   ├── cgroup.procs
│   ├── notify_on_release
│   └── tasks
├── cgroup-2
│   ├── cgroup.clone_children
│   ├── cgroup.procs
│   ├── notify_on_release
│   └── tasks
├── cgroup.clone_children
├── cgroup.procs
├── cgroup.sane_behavior
├── notify_on_release
├── release_agent
└── tasks
```

上面这些文件就是`hierarchy`中`cgroup`根节点的配置项,这些文件的含义是:

- `group.clone_children` `cpuset`的`subsystem`会读取这个配置文件,如果这个值(默认值是0)是 **1** 子`cgroup`才会继承父`cgroup`的`cpuset`的配置

- `cgroup.procs`是树中当前节点 `cgroup` 中的进程组ID,现在的位置是根节点,这个文件中会有现在系统中所有进程组的ID (查看目前全部进程PID `ps -ef | awk '{print $2}'`)

- `notify_on_release` 标志当这个`cgroup`最后一个进程退出的时候是否执行了`release_agent` (`notify_on_release`和`release_agent` 会一起使用)

- `release_agent` 则是一个路径,通常用作进程退出后自动清理不再使用的`cgroup`

- `task` 标识该`cgroup`下面进程ID,如果把一个进程ID写到`task`文件中,便会把相应的进程加入到这个`cgroup`中

在刚刚创建好的`hierarchy`上`cgroup`根节点中扩展出两个子`cgroup`, 它们会继承父`cgroup`的属性

#### 通过subsystem限制cgroup中进程的资源

在上面创建的 hierarchy`并没有关联到任何的`subsystem， 需要我们手动创建`subsystem`挂载上去。

```shell
mkdir -p mycgroup/memory && mount -t cgroup -o memory mycgroup-memory `pwd`/mycgroup/memory
```

执行完上面命令后在memory会生成很多文件(这些文件其实是继承至/sys/fs/cgroup/memory), 其实不需要我们手动挂载cgroup，下面将演示如何使用cgroup进行资源限制~

### 如何使用cgroup进行资源限制

cgroups的创建很简单，只需要在相应的子系统下创建目录即可(默认是在/sys/fs/cgroup目录下), 接下来将演示如何限制cpu跟内存使用数量

```
[root@node1 cgroup]# ls
blkio  cpuacct      cpuset   freezer  memory   net_cls,net_prio  perf_event  rdma
cpu    cpu,cpuacct  devices  hugetlb  net_cls  net_prio          pids        systemd
```

#### CPU 子系统
CPU子系统有两个目录, cpuset和cpu,cpuacct, 其中cpu,cpuacct用于设置cpu配额，cpuset用于设置cpu绑定(设置进程只能在指定的核上), 下面对cpu 子系统的一些字段进行说明：

- `cpu.cfs_period_us`: 表示一个cpu带宽，单位为微秒(默认是10000000)。系统总CPU带宽： cpu核心数 * cfs_period_us。

- `cpu.cfs_quota_us` 文件: 代表在某一个阶段限制的 CPU 时间总量，单位为微秒。cfs_quota_us为-1，表示使用的CPU不受cgroup限制。cfs_quota_us的最小值为1ms(1000)，最大值为1s。 结合cfs_period_us可以限制进程使用的cpu。例如配置cfs_period_us=10000，而cfs_quota_us=20000。那么该进程就可以可以用2个cpu core。

- `cpuacct.stat`: 记录cgroup的所有任务（包括其子孙层级中的所有任务）使用的用户和系统CPU时间.

- `cpuacct.usage`: 记录这​​​个​​​cgroup中​​​所​​​有​​​任​​​务​​​（包括其子孙层级中的所有任务）消​​​耗​​​的​​​总​​​CPU时​​​间​​​（纳​​​秒​​​）。

- `cpuset.cpus`: 指​​​定​​​允​​​许​​​这​​​个​​​ cgroup 中​​​任​​​务​​​(进程)访​​​问​​​的​​​ CPU。​​​这​​​是​​​一​​​个​​​用​​​逗​​​号​​​分​​​开​​​的​​​列​​​表​​​，格​​​式​​​为​​​ ASCII，使​​​用​​​小​​​横​​​线​​​（"-"）代​​​表​​​范​​​围​​​。​​​如下，代​​​表​​​ CPU 0、​​​1、​​​2 和​​​ 16。​​​

- `cpuset.mems`: 指​​​定​​​允​​​许​​​这​​​个​​​ cgroup 中​​​任​​​务​​​可​​​访​​​问​​​的​​​内​​​存​​​节​​​点​​​。​​​这​​​是​​​一​​​个​​​用​​​逗​​​号​​​分​​​开​​​的​​​列​​​表​​​，格​​​式​​​为​​​ ASCII，使​​​用​​​小​​​横​​​线​​​（"-"）代​​​表​​​范​​​围​​​。​​​如下代​​​表​​​内​​​存​​​节​​​点​​​ 0、​​​1、​​​2 和​​​ 16。

![cpu子系统](https://tva1.sinaimg.cn/large/008i3skNly1gtsuo2k4u5j615k0u07ag02.jpg)

1. **创建CPU Cgroup** 

创建cpu子系统很简单，只需要在cpu子系统下创建一个目录即可。

```shell
mkdir /sys/fs/cgroup/cpu/mydocker
```

执行完上述命令后，我们查看一下我们新创建的目录下发生了什么？

```
[root@node1 mydocker]# ls -l /sys/fs/cgroup/cpu/mydocker
total 0
-rw-r--r--. 1 root root 0 Aug 24 10:20 cgroup.clone_children
-rw-r--r--. 1 root root 0 Aug 24 10:20 cgroup.procs
-r--r--r--. 1 root root 0 Aug 24 10:20 cpuacct.stat
-rw-r--r--. 1 root root 0 Aug 24 10:20 cpuacct.usage
-r--r--r--. 1 root root 0 Aug 24 10:20 cpuacct.usage_all
-r--r--r--. 1 root root 0 Aug 24 10:20 cpuacct.usage_percpu
-r--r--r--. 1 root root 0 Aug 24 10:20 cpuacct.usage_percpu_sys
-r--r--r--. 1 root root 0 Aug 24 10:20 cpuacct.usage_percpu_user
-r--r--r--. 1 root root 0 Aug 24 10:20 cpuacct.usage_sys
-r--r--r--. 1 root root 0 Aug 24 10:20 cpuacct.usage_user
-rw-r--r--. 1 root root 0 Aug 24 10:20 cpu.cfs_period_us
-rw-r--r--. 1 root root 0 Aug 24 08:52 cpu.cfs_quota_us
-rw-r--r--. 1 root root 0 Aug 24 10:20 cpu.rt_period_us
-rw-r--r--. 1 root root 0 Aug 24 10:20 cpu.rt_runtime_us
-rw-r--r--. 1 root root 0 Aug 24 10:20 cpu.shares
-r--r--r--. 1 root root 0 Aug 24 10:20 cpu.stat
-rw-r--r--. 1 root root 0 Aug 24 10:20 notify_on_release
-rw-r--r--. 1 root root 0 Aug 24 08:54 tasks
```

由上可以看到我们新建的目录下被自动创建了很多文件(（这里利用到了继承，子进程会继承父进程的 cgroup）)


2. **创建进程，加入 cgroup**

这里为了方便演示，我先把当前运行的 shell 进程加入 cgroup，然后在当前 shell 运行 cpu 耗时任务（这里利用到了继承，子进程会继承父进程的 cgroup）。
```
cd /sys/fs/cgroup/cpu/mydocker

echo $$ > tasks

echo $$
```

3. **设置cpu quote配额**

```
# 设置当前shell 可以使用1core cpu
echo 1000000 > cpu.cpu.cfs_period_us
echo 1000000 > cpu.cfs_quota_us
```

4. **观察cgroup是否生效**

```shell
# 当前shell

while true; do echo; done;

# 另起一个shell
top -p pid(刚才的$$的数字)
```

通过top命令可以看到刚才的shell 进程cpu已经达到了100%，说明cgroup起作用了.

![cgroup](https://tva1.sinaimg.cn/large/008i3skNly1gtsvbodpddj61io0nmdk102.jpg)

让我们更近一步，设置bash进程可以使用的cpu为0.5core, 通过`echo 500000 > cpu.cfs_quota_us` 然后继续观察，发现cpu到达50%后就上不去了。验证完 cgroup 限制 cpu，我们使用相似的方法来验证 cgroup 对内存的限制。

![cgroup](https://tva1.sinaimg.cn/large/008i3skNly1gtsvdt0t6lj61ju0osdjx02.jpg)

#### Memory 子系统

创建memory子系统的方式跟cpu子系统的方式差不多，只需要在memory子系统下创建一个目录即可。

1. **在 memory 子系统下创建 cgroup**

```
mkdir /sys/fs/cgroup/memory/mydocker
```

执行完上述命令后，我们查看一下我们新创建的目录下发生了什么？

```
[root@node1 mydocker]# ls -l /sys/fs/cgroup/memory/mydocker
total 0
-rw-r--r--. 1 root root 0 Aug 24 23:34 cgroup.clone_children
--w--w--w-. 1 root root 0 Aug 24 23:34 cgroup.event_control
-rw-r--r--. 1 root root 0 Aug 24 23:34 cgroup.procs
-rw-r--r--. 1 root root 0 Aug 24 23:34 memory.failcnt
--w-------. 1 root root 0 Aug 24 23:34 memory.force_empty
-rw-r--r--. 1 root root 0 Aug 24 23:34 memory.kmem.failcnt
-rw-r--r--. 1 root root 0 Aug 24 23:34 memory.kmem.limit_in_bytes
-rw-r--r--. 1 root root 0 Aug 24 23:34 memory.kmem.max_usage_in_bytes
-r--r--r--. 1 root root 0 Aug 24 23:34 memory.kmem.slabinfo
-rw-r--r--. 1 root root 0 Aug 24 23:34 memory.kmem.tcp.failcnt
-rw-r--r--. 1 root root 0 Aug 24 23:34 memory.kmem.tcp.limit_in_bytes
-rw-r--r--. 1 root root 0 Aug 24 23:34 memory.kmem.tcp.max_usage_in_bytes
-r--r--r--. 1 root root 0 Aug 24 23:34 memory.kmem.tcp.usage_in_bytes
-r--r--r--. 1 root root 0 Aug 24 23:34 memory.kmem.usage_in_bytes
-rw-r--r--. 1 root root 0 Aug 24 23:34 memory.limit_in_bytes
-rw-r--r--. 1 root root 0 Aug 24 23:34 memory.max_usage_in_bytes
-rw-r--r--. 1 root root 0 Aug 24 23:34 memory.memsw.failcnt
-rw-r--r--. 1 root root 0 Aug 24 23:34 memory.memsw.limit_in_bytes
-rw-r--r--. 1 root root 0 Aug 24 23:34 memory.memsw.max_usage_in_bytes
-r--r--r--. 1 root root 0 Aug 24 23:34 memory.memsw.usage_in_bytes
-rw-r--r--. 1 root root 0 Aug 24 23:34 memory.move_charge_at_immigrate
-r--r--r--. 1 root root 0 Aug 24 23:34 memory.numa_stat
-rw-r--r--. 1 root root 0 Aug 24 23:34 memory.oom_control
----------. 1 root root 0 Aug 24 23:34 memory.pressure_level
-rw-r--r--. 1 root root 0 Aug 24 23:34 memory.soft_limit_in_bytes
-r--r--r--. 1 root root 0 Aug 24 23:34 memory.stat
-rw-r--r--. 1 root root 0 Aug 24 23:34 memory.swappiness
-r--r--r--. 1 root root 0 Aug 24 23:34 memory.usage_in_bytes
-rw-r--r--. 1 root root 0 Aug 24 23:34 memory.use_hierarchy
-rw-r--r--. 1 root root 0 Aug 24 23:34 notify_on_release
-rw-r--r--. 1 root root 0 Aug 24 23:34 tasks
```

由上可以看到我们新建的目录下被自动创建了很多文件(这里利用到了继承，子进程会继承父进程的 cgroup), 其中 memory.limit_in_bytes 文件代表内存使用总量，单位为 byte。例如，这里我希望对内存使用限制为 1G(1G = 1024*1024*1024)，则向 memory.limit_in_bytes 文件写入 1073741824即可

2. **创建进程，加入 cgroup**

```
cd /sys/fs/cgroup/memory/mydocker

echo $$ > tasks

echo $$
```

3. **设置memory 配额**

```
# 设置当前shell 可以使用1G内存
echo 1073741824 > memory.limit_in_bytes
```

4. **观察cgroup是否生效**

```
# 使用stress-ng进行压力测试
stress-ng --vm 5 --vm-bytes 250M
```

通过运行stress-ng压力测试后，能发现终端会直接被卡死(其实无法说明Cgroup是否生效， 使用pidstat也不好查看内存占用情况)

### pids子系统




### 参考文档

- [使用cgroups控制进程cpu配额](https://www.jianshu.com/p/66734cde7994)               //图不错
- [一文彻底搞懂Linux Cgroup如何限制容器资源](https://juejin.cn/post/6844904079076884493) 	 //可以看看
- [资源限制：如何通过 Cgroups 机制实现资源限制？](http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/%E7%94%B1%E6%B5%85%E5%85%A5%E6%B7%B1%E5%90%83%E9%80%8F%20Docker-%E5%AE%8C/10%20%20%E8%B5%84%E6%BA%90%E9%99%90%E5%88%B6%EF%BC%9A%E5%A6%82%E4%BD%95%E9%80%9A%E8%BF%87%20Cgroups%20%E6%9C%BA%E5%88%B6%E5%AE%9E%E7%8E%B0%E8%B5%84%E6%BA%90%E9%99%90%E5%88%B6%EF%BC%9F.md)       //推荐动手实验
- [Cgroup中的CPU资源控制](https://zhuanlan.zhihu.com/p/346050404)       //cgroup各个字段的含义
