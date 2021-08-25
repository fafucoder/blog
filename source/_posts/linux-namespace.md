---
title: linux namespace技术
date: 2020-05-09 22:20:30
tags:
- linux
categories:
- linux
---

### 概述

Namespace是对全局系统资源的一种封装隔离，使得处于不同namespace的进程拥有独立的全局系统资源，改变一个namespace中的系统资源只会影响当前namespace里的进程，对其他namespace中的进程没有影响。

linux内核支持的namespace如下所示：
```
名称        宏定义             隔离内容                       
Cgroup      CLONE_NEWCGROUP   Cgroup root directory (since Linux 4.6)
IPC         CLONE_NEWIPC      System V IPC, POSIX message queues (since Linux 2.6.19)
Network     CLONE_NEWNET      Network devices, stacks, ports, etc. (since Linux 2.6.24)
Mount       CLONE_NEWNS       Mount points (since Linux 2.4.19)
PID         CLONE_NEWPID      Process IDs (since Linux 2.6.24)
User        CLONE_NEWUSER     User and group IDs (started in Linux 2.6.23 and completed in Linux 3.8)
UTS         CLONE_NEWUTS      Hostname and NIS domain name (since Linux 2.6.19)
```
1. Mount（`mnt`） 隔离挂载点
2. Process ID (`pid`) 隔离进程 ID
3. Network (`net`) 隔离网络设备、网络栈、端口等等
4. Interprocess Communication (`ipc`)  隔离信号量、消息队列和共享内存
5. UTS Namespace(`uts`) 隔离主机名和域名
6. User Namespace (`user`)  隔离用户和用户组

注意:

- 由于Cgroup namespace在4.6的内核中才实现，并且和cgroup v2关系密切，现在普及程度还不高，比如docker现在就还没有用它，所以在namespace这个系列中不会介绍Cgroup namespace。
- 各命名空间的详细信息 [参考这里](https://man7.org/linux/man-pages/man7)

### namespace系统调用接口
在[namespace man](https://man7.org/linux/man-pages/man7/namespaces.7.html)上提供的系统调用包括clone, setns, unshare, ioctl

1. clone: 创建一个新的进程并把他放到新的namespace中
``` 
int clone(int (*child_func)(void *), void *child_stack, int flags, void *arg);
```

2. setns： 将当前进程加入到已有的namespace中, (ip netns命令调用的就是setns系统调用)
```
int setns(int fd, int nstype);
```

3. unshare: 使当前进程退出指定类型的namespace，并加入到新创建的namespace（unshare命令）
```
int unshare(int flags);
```

clone和unshare的区别
- unshare是使当前进程加入新的namespace
- clone是创建一个新的子进程，然后让子进程加入新的namespace，而当前进程保持不变

### 查看进程所属的namespace
系统中的每个进程都有/proc/[pid]/ns/这样一个目录，里面包含了这个进程所属namespace的信息
```
dev@ubuntu:~$ ls -l /proc/$$/ns     
total 0
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 ipc -> ipc:[4026531839]   
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 mnt -> mnt:[4026531840]   
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 net -> net:[4026531957]
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 pid -> pid:[4026531836]
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 user -> user:[4026531837]
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 uts -> uts:[4026531838]
```

### uts namespace

uts(UNIX Time-Sharing System) namespace可隔离hostname和NIS Domain name资源，使得一个宿主机可拥有多个主机名或Domain Name。换句话说，可让不同namespace中的进程看到不同的主机名。

```bash
# 设置当前root namespace的主机名为hostname
[root@master ~]# hostname hello
[root@master ~]# hostname
hello

# -u或--uts表示创建一个uts namespace 进入了新的namespace中的shell
[root@master ~]# unshare -u /bin/sh

# 其主机名初始时也是hello, 其拷贝自上级namespace资源
sh-4.2# hostname
hello
sh-4.2# hostname world
sh-4.2# hostname
world
sh-4.2# exit
exit
[root@master ~]# hostname
hello
```

通过pstree 查看进程树 `sshd(28352)---bash(28375)---bash(4146)-+-grep(4434)`

![pstree](https://tva1.sinaimg.cn/large/008i3skNgy1gslihlpatvj31gm0s2jyw.jpg)

原理: Bash进程(28375)运行在当前namespace中，它将fork一个新进程来运行unshare程序，unshare程序加载完成后，将创建一个新的uts namespace，unshare进程自身将加入到这个uts namespace中，unshare进程内部再exec加载/bin/bash，于是unshare进程被替换为/bin/bash进程，/bin/bash进程也将运行在uts namespace中

ps: 当namespace中的/bin/sh进程退出，如果namespace中没有任何进程，该namespace将自动销毁。注意，在默认情况下，namespace中必须要有至少一个进程，否则将被自动被销毁。但也有一些手段可以让namespace持久化，即使已经没有任何进程在其中运行。(例如创建一个文件)

### mount namespace

mount namespace可隔离出一个具有独立挂载点信息的运行环境，内核知道如何去维护每个namespace的挂载点列表。即每个namespace之间的挂载点列表是独立的，各自挂载互不影响。(用户通常使用mount命令来挂载普通文件系统，但实际上mount能挂载的东西非常多，甚至连现在功能完善的Linux系统，其内核的正常运行也都依赖于挂载功能，比如挂载根文件系统/。其实所有的挂载功能和挂载信息都由内核负责提供和维护，mount命令只是发起了mount()系统调用去请求内核。)

内核将每个进程的挂载点信息保存在`/proc/<pid>/{mountinfo,mounts,mountstats}`三个文件中:

```shell
[root@node1 ~]# ls -l /proc/$$/mount*
-r--r--r--. 1 root root 0 Aug 16 08:47 /proc/5844/mountinfo
-r--r--r--. 1 root root 0 Aug 16 08:47 /proc/5844/mounts
-r--------. 1 root root 0 Aug 16 08:47 /proc/5844/mountstats
```

具有独立的挂载点信息，意味着每个mnt namespace可具有独立的目录层次，这在容器中起了很大作用：`容器可以挂载只属于自己的文件系统。`

当创建mount namespace时，内核将拷贝一份当前namespace的挂载点信息列表到新的mnt namespace中，此后两个mnt namespace就没有了任何关系.(也不是完全没关系，有个shared subtrees选项可设置挂载共享方式，默认是私有的，也就是没关系)

```shell
#! /bin/bash

# 创建目录
mkdir -p iso/iso1/dir1 && mkdir -p iso/iso2/dir2

# 生成iso文件
cd iso && mkisofs -o 1.iso iso1 && mkisofs -o 2.iso iso2

# 挂载iso1
mkdir /mnt/{iso1,iso2} && mount 1.iso /mnt/iso1 && mount | grep iso1

# 创建mount+uts namespace
unshare -m -u /bin/bash

# 在namespace中挂载iso2
mount 2.iso2 /mnt/iso2/

# 查看挂载信息
echo "查看挂载信息\n\n"
mount | grep 'iso[12]'

# 卸载挂载, 此时看到没有iso1的挂载信息
umount /mnt/iso1/

echo "查看挂载信息\n\n"
mount | grep 'iso[12]'

# 新起一个terminal, 查看挂载信息没有iso2的挂载信息，但是还是有iso1的挂载信息
mount | grep 'ios[12]'

```

### pid namespace

PID namespace表示隔离一个具有独立PID的运行环境。在每一个pid namespace中，进程的pid都从1开始，且和其他pid namespace中的PID互不影响。这意味着，不同pid namespace中可以有相同的PID值。

因为PID namespace中的PID是独立的，每一个PID namespace都允许一些特殊的操作：允许pid namespace挂起、迁移以及恢复，就像虚拟机一样。

### user namespace



### network namespace

network namespace用来隔离网络环境，在network namespace中，网络设备、端口、套接字、网络协议栈、路由表、防火墙规则等都是独立的。

因为network namespace中具有独立的网络协议栈，因此每个network namespace中都有一个lo接口，但lo接口默认未启动，需要手动启动起来。

让某个network namespace和root network namespace或其他network namespace之间保持通信是一个非常常见的需求，这一般通过veth虚拟设备实现。veth类型的虚拟设备由一对虚拟的eth网卡设备组成，像管道一样，一端写入的数据总会从另一端流出，从一端读取的数据一定来自另一端。

用户可以将veth的其中一端放在某个network namespace中，另一端保留在root network namespace中。这样就可以让用户创建的network namespace和宿主机通信。

(实验部分略，之前有相关的实验了)

### unshare 命令

> 操作namespace的命令有unshare, lsns nsenter

unshare命名就是unshare系统调用的实现，下面将通过unshare命令演示namespace的隔离技术

![unshare 命令](https://tva1.sinaimg.cn/large/008i3skNgy1gslhd4myd7j31g00o40wh.jpg)

### lsns 命令

lsns命令列举出当前已经创建的namespace

![lsns](https://tva1.sinaimg.cn/large/008i3skNgy1gslhmuhvacj31k60tojv7.jpg)

### 参考文档

- https://medium.com/@teddyking/linux-namespaces-850489d3ccf   //namespace 概念
- https://github.com/teddyking/ns-process  //namespace 源代码
- https://coolshell.cn/articles/17010.html //酷壳 DOCKER基础技术
- https://segmentfault.com/a/1190000006908272 //Namespace概述
- https://www.junmajinlong.com/virtual/namespace/ns_overview/  //各个namespace详解

