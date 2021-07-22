---
title: linux strace命令
date: 2021-07-19 00:13:28
tags:
- linux
categories:
- linux
---

### 简述

在Linux世界，进程不能直接访问硬件设备，当进程需要访问硬件设备(比如读取磁盘文件，接收网络数据等等)时，必须由用户态模式切换至内核态模式，通过系统调用访问硬件设备。strace是一个可用于诊断、调试和教学的Linux用户空间跟踪器。我们用它来监控用户空间进程和内核的交互，比如系统调用、信号传递、进程状态变更等。

> strace底层使用内核的ptrace特性来实现其功能。

在Linux系统上，进程通过glibc库封装的函数，间接使用系统调用。Linux内核目前有300多个系统调用，详细的列表可以通过syscalls手册页查看。这些系统调用主要分为几类：

```
文件和设备访问类: 比如open/close/read/write/chmod等
进程管理类: fork/clone/execve/exit/getpid等
信号类: signal/sigaction/kill 等
内存管理: brk/mmap/mlock等
进程间通信IPC: shmget/semget * 信号量，共享内存，消息队列等
网络通信: socket/connect/sendto/sendmsg 等
```

strace有两种运行模式：

- 通过它启动要跟踪的进程， 只需要在原本的命令前加上strace即可, 例如： strace ls /usr/local
- 通过它跟踪已经运行的进程，在不中断进程执行的情况下诊断进程，只需要指定-p pid 即可。例如: strace -p 1234

### 参数

```
-c   统计每一系统调用的所执行的时间,次数和出错的次数等. 
-d   输出strace关于标准错误的调试信息. 
-f   跟踪由fork调用所产生的子进程. 
-ff  如果提供-o filename,则所有进程的跟踪结果输出到相应的filename.pid中,pid是各进程的进程号. 
-F   尝试跟踪vfork调用.在-f时,vfork不被跟踪. 
-i   输出系统调用的入口指针. 
-q   禁止输出关于脱离的消息. 
-r   打印出相对时间关于每一个系统调用. 
-t   在输出中的每一行前加上时间信息. 
-tt  在输出中的每一行前加上时间信息,微秒级. 
-ttt 微秒级输出,以秒了表示时间. 
-T   显示每一调用所耗的时间. 
-v   输出所有的系统调用.一些调用关于环境变量,状态,输入输出等调用由于使用频繁,默认不输出. 
-V   输出strace的版本信息. 
-x   以十六进制形式输出非标准字符串 
-xx  所有字符串以十六进制形式输出. 
-e expr 指定一个表达式,用来控制如何跟踪 
-o filename 将输出写入文件filename 
-p pid 跟踪指定的进程pid. 
-s strsize 指定输出的字符串的最大长度.默认为32
-u username 以username 的UID和GID执行被跟踪的命令
```

> strace -e expr格式如下 `[qualifier=][!]value1[,value2]...  `qualifier只能是 trace,abbrev,verbose,raw,signal,read,write其中之一, value是用来限定的符号或数字。默认的 qualifier是 trace。 例如: -eopen等价于 -e trace=open,表示只跟踪open调用.而-etrace!=open表示跟踪除了open以外的其他调用.有两个特殊的符号 all 和 none.
>
>  如果人工输入每一个具体的系统调用名称，可能容易遗漏。于是strace提供了几类常用的系统调用组合名字。
>
> -e trace=file          跟踪和文件访问相关的调用(参数中有文件名)
> -e trace=process  和进程管理相关的调用，比如fork/exec/exit_group
> -e trace=network  和网络通信相关的调用，比如socket/sendto/connect
> -e trace=signal      信号发送和处理相关，比如kill/sigaction
> -e trace=desc       和文件描述符相关，比如write/read/select/epoll等
> -e trace=ipc          进程间通信相关，比如shmget等

