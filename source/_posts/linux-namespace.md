---
title: linux namespace技术
date: 2020-05-09 22:20:30
tags:
- linux
categories:
- linux
---

### 概述

​	Namespace是对全局系统资源的一种封装隔离，使得处于不同namespace的进程拥有独立的全局系统资源，改变一个namespace中的系统资源只会影响当前namespace里的进程，对其他namespace中的进程没有影响。

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

![pstree](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNgy1gslihlpatvj31gm0s2jyw.jpg)

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

pid namespace表示隔离一个具有独立PID的运行环境。在每一个pid namespace中，进程的pid都从1开始，且和其他pid namespace中的PID互不影响。这意味着，不同pid namespace中可以有相同的PID值。

因为pid namespace中的PID是独立的，每一个PID namespace都允许一些特殊的操作：允许pid namespace挂起、迁移以及恢复，就像虚拟机一样。

在介绍pid namespace之前，先创建其他类型的namespace然后查看进程关系:

```shell
# 在root namespace中查看当前进程的进程ID
echo $$

# 创建一个uts namespace
unshare -u /bin/bash

# 在uts namespace中查看当前进程
pstree -p | grep grep
```

运行截图如下所示， unshare进程会在创建新的namespace后会被改namespace中的第一个进程给替换掉。

![进程关系](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gtu7oa7m1hj613s0500tb02.jpg)

创建新的pid namespace方式：

```shel
# unshare --pid --fork [--mount-proc] <CMD>
# 	--pid(-p):    表示创建pid namespace
# 	--mount-proc: 表示创建pid namespace时重新挂载procfs
# 	--fork(-f):   表示创建pid namespace时，不是直接替换unshare进程，而是fork unshare进程，并使用CMD替换fork出来的子进程
```

使用--fork的操作的结果是(如下图): unshare进程被保留，且保留在原来的pid namespace中，而不是加入到新的pid namespace中。

![pid](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gtu8axwk5vj61so08iwh202.jpg)

#### pid namespace 嵌套

pid namespace可以存在嵌套关系。所有的子孙pid namespace中的进程信息都会保存在父级以及祖先级namespace中，只不过在不同嵌套层级中，同一个进程对应的PID不同。

```shell
# ns1 namespace
[root@node1 ~]# unshare -pmfu --mount-proc /bin/bash
[root@node1 ~]# hostname ns1
[root@node1 ~]# exec bash

# ns2 namespace
[root@ns1 ~]# unshare -pmfu --mount-proc /bin/bash
[root@ns1 ~]# hostname ns2
[root@ns1 ~]# exec bash
[root@ns2 ~]# pstree -p | grep grep
bash(1)-+-grep(27)

# root namespace 
[root@node1 ~]# pstree -p 19288
bash(19288)───unshare(14472)───bash(14473)───unshare(24783)───bash(24786)
```

#### pid namespace和procfs

/proc目录是内核对外暴露的可供用户查看或修改的内核中所记录的信息，包括内核自身的部分信息以及每个进程的信息。比如对于pid=N的进程来说，它的信息保存在/proc/<N>目录下。在操作系统启动的过程中，会挂载procfs到/proc目录，它存在于root namespace中。

但是，创建新的pid namespace时不会自动重新挂载procfs，而是直接拷贝父级namespace的挂载点信息。这使得在新的pid namespace中仍然保留了父级namespace的/proc目录，也就是在新创建的这个pid namespace中仍然保留了父级的进程信息。

```shell
# 注意没--mount-proc
[root@node1 ~]# unshare -pmuf /bin/bash
[root@node1 ~]# hostname ns1
[root@node1 ~]# exec bash

# 查看进程树， 发现还是宿主机的
[root@ns1 ~]# pstree
systemd─┬─NetworkManager─┬─3*[dhclient]
        │                └─3*[{NetworkManager}]
        ├─agetty
        ├─auditd───{auditd}
        ├─chronyd
 ....
```

之所以有上述问题，其原因是在pid namespace中保留了root namespace中的/proc目录，而不是属于pid namespace自己的/proc。

但用户创建pid namespace时希望的是有完全独立的进程运行环境。这时，需要在pid namespace中重新挂载procfs，或者在创建pid namespace时指定--mount-proc选项。

```shell
# 注意这儿添加了mount-proc参数
[root@node1 ~]# unshare -pmuf --mount-proc /bin/bash
[root@node1 ~]# hostname ns1
[root@node1 ~]# exec bash

# 进程树中展示了bash 为init进程
[root@ns1 ~]# pstree -p
bash(1)───pstree(24)
```

#### pid namespace 信号量问题

pid=1的进程是每一个pid namespace的核心进程(init进程)，它不仅负责收养其所在pid namespace中的孤儿进程，还影响整个pid namespace。

当pid namespace中pid=1的进程退出或终止，内核默认会发送SIGKILL信号给该pid namespace中的所有进程以便杀掉它们(如果该pid namespace中有子孙namespace，也会直接被杀)。

在创建pid namespace时可以通过--kill-child选项指定pid=1的进程终止后内核要发送给pid namespace中进程的信号，其默认信号便是SIGKILL。

```shell
unshare -p -f -m -u --mount-proc --kill-child=SIGHUP /bin/bash
```

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

![unshare 命令](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNgy1gslhd4myd7j31g00o40wh.jpg)

### lsns 命令

lsns命令列举出当前已经创建的namespace

![lsns](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNgy1gslhmuhvacj31k60tojv7.jpg)

### 参考文档

- https://medium.com/@teddyking/linux-namespaces-850489d3ccf   //namespace 概念
- https://github.com/teddyking/ns-process  //namespace 源代码
- https://coolshell.cn/articles/17010.html //酷壳 DOCKER基础技术
- https://segmentfault.com/a/1190000006908272 //Namespace概述
- https://www.junmajinlong.com/virtual/namespace/ns_overview/  //各个namespace详解

