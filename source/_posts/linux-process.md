---
title: linux中进程，线程，协程
date: 2021-03-06 23:47:19
tags:
- linux
categories:
- linux
---

### 进程， 线程，协程



### 孤儿进程，僵尸进程，守护进程



### systemd进程

**提到 systemd 不得不提一下 Linux 的启动流程，这样才能清楚 systemd 在 Linux 系统中的地位和作用**

#### linux 启动流程

1. 加电自检(检查硬件是否有问题)
2. GRUB引导
3. 内核加载
4. init 初始化(内核启动第一个**用户空间应用程序**，即 systemd 进程， 通过dmesg能查看)

![dmesg](https://tva1.sinaimg.cn/large/008eGmZEly1goanfk29j1j328m0tcqbm.jpg)

#### systemd简介

*systemd* 是一个 Linux 系统基础组件的集合，提供了一个系统和服务管理器，运行为 PID 1 并负责启动其它程序, 所有的进程都会被挂在这个进程下，如果这个进程退出了，那么所有的进程都被 kill 。 systemd功能包括：

- 支持并行化任务；
- 同时采用 socket 式与 [D-Bus](https://wiki.archlinux.org/index.php/D-Bus_(简体中文)) 总线式激活服务；
- 按需启动守护进程（daemon）；
- 利用 Linux 的 [cgroups](https://wiki.archlinux.org/index.php/Cgroups) 监视进程；
- 支持快照和系统恢复；
- 维护挂载点和自动挂载点；
- 各服务间基于依赖关系进行精密控制。
- systemd* 支持 SysV 和 LSB 初始脚本，可以替代 sysvinit。
- 除此之外，功能还包括日志进程、控制基础系统配置，维护登陆用户列表以及系统账户、运行时目录和设置，可以运行容器和虚拟机，可以简单的管理网络配置、网络时间同步、日志转发和名称解析等。

通过pstree 能够查看进程数状态，用户空间的进程都挂在 PID 为 1 的 systemd 进程下。(似乎systemd进程无法被杀死，kill -9 1似乎无效！)

![pstree 进程数](https://tva1.sinaimg.cn/large/008eGmZEly1goan63vhgqj313s0pygp3.jpg)

#### systemd 体系架构

![systemd体系架构](https://tva1.sinaimg.cn/large/008eGmZEly1goanaecdizj31g40qc1kx.jpg)

![systemd交互](https://tva1.sinaimg.cn/large/008eGmZEly1goannznnthj31bn0u07wh.jpg)

- 最底层：systemd 内核层面依赖 cgroup、autofs、kdbus

- 第二层：systemd libraries 是 systemd 依赖库

- 第三层：systemd Core 是 systemd 自己的库

- 第四层：systemd daemons 以及 targets 是自带的一些基本 unit、target，类似于 sysvinit 中自带的脚本

- 最上层就是和 systemd 交互的一些工具其中：

  1. systemctl 命令控制 `systemd` 的管理系统和服务的命令行工具

  2. journalctl 命令查詢 `systemd` 日志系统

  3. loginctl 命令控制 `systemd` 登入管理器

  4. systemd-analyze 分析系统启动效能

  ![systemd命令大全](https://tva1.sinaimg.cn/large/008eGmZEly1goanl9xhdsj320c08etap.jpg)

#### systemd软连接

当我们使用 reboot 、poweroff 、shutdown 等命令的时候，其实并不是执行该命令本身，背后是调用的 systemctl 命令。systemctl 命令会将 reboot 这些命令作为 $1 参数传递进去。所以执行 reboot 和 systemctl reboot 本质上是一样的。

![systemd软连接](https://tva1.sinaimg.cn/large/008eGmZEly1goanp7lgg9j319q07o75f.jpg)

![systemd system commands](https://tva1.sinaimg.cn/large/008eGmZEly1goansqy13wj317y0ewdib.jpg)

### 参考文档

- [Linux 的小伙伴 systemd 详解](https://blog.k8s.li/systemd.html)   //木子的博客
- [LINUX PID 1 和 SYSTEMD](https://coolshell.cn/articles/17998.html)  //酷壳