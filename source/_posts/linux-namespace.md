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
注意:

- 由于Cgroup namespace在4.6的内核中才实现，并且和cgroup v2关系密切，现在普及程度还不高，比如docker现在就还没有用它，所以在namespace这个系列中不会介绍Cgroup namespace。
- 各命名空间的[详细信息](https://man7.org/linux/man-pages/man7)可以搜寻到

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

### 插件进程所属的namespace
系统中的每个进程都有/proc/[pid]/ns/这样一个目录，里面包含了这个进程所属namespace的信息
```
dev@ubuntu:~$ ls -l /proc/$$/ns     
total 0
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 cgroup -> cgroup:[4026531835] #(since Linux 4.6)
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 ipc -> ipc:[4026531839]       #(since Linux 3.0)
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 mnt -> mnt:[4026531840]       #(since Linux 3.8)
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 net -> net:[4026531957]       #(since Linux 3.0)
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 pid -> pid:[4026531836]       #(since Linux 3.8)
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 user -> user:[4026531837]     #(since Linux 3.8)
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 uts -> uts:[4026531838]       #(since Linux 3.0)
```

### 参考文档
- https://medium.com/@teddyking/linux-namespaces-850489d3ccf   //namespace 概念
- https://github.com/teddyking/ns-process  //namespace 源代码
- https://coolshell.cn/articles/17010.html //酷壳 DOCKER基础技术
- https://segmentfault.com/a/1190000006908272 //Namespace概述

