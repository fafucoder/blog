---
stitle: linux ps命令和 pstree 命令
date: 2021-03-25 22:50:07
tags:
- linux
categories:
- linux
---

### 概述

ps命令能够给出当前系统中进程的快照。它能捕获系统在某一事件的进程状态。如果你想不断更新查看的这个状态，可以使用top命令。

ps 命令支持的语法风格

1. **UNIX风格**: 选项可以组合在一起，并且选项前必须有“-”连字符
2. **BSD风格**: 选项可以组合在一起，但是选项前不能有“-”连字符
3. **GNU风格**: 选项前有两个“-”连字符

![example](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp1xlxnfb4j31be0u078m.jpg)

### 参数

--sort: 对结果集进行排序

-L: 显示指定进程的线程, 例如 ps -L 1234(1234 为pid)

-o: 控制输入的列数据

-e: 显示所有进程信息

-f: 做一个更为完整的输出,通常跟`-e`参数一块使用

-u: 以用户为主的进程状态,通常跟-a一块使用

-a: 显示现行终端机下的所有进程，包括其他用户的进程(这是当前终端的所有进程，通过配合`-aux`查看所有的进程)

-x: 显示较完整信息。通常与` -a `这个参数一起使用

-A: 列出所有的进程(这是所有进程)

-C: 根据进程名做筛选, 例如， ps -C nginx

-j: 以job的格式展示

##### ps -aux 与 ps -ef的区别

输出的格式不一样，ps -aux输出包括进程的cpu跟内存的使用情况，而ps -ef只输出进程的信息(如pid, ppid等)，未输出进程的更详细信息，如果要通过`--sort`参数做排序的话，通常使用`ps -aux`

##### --sort参数详解

在`--sort`参数中，`-`减号是降序，`+`号表示升序，--sort的排序列跟`-o` 的列一致，默认是升序格式

##### -o 格式化的参数

format格式较多，可通过man ps查看，主要的参数有`pid, ppid, pgid, cmd(command,agrs), pcpu, pmem, tty `等

![ps -o](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp1zdwx9ukj31830u0n6o.jpg)

#### 显示进程树

```shell
ps -ejH 或者 ps axjf
```

![显示进程数](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp1zjd7r6vj31q40u0dp6.jpg)

#### 显示线程

```shell
ps -aux | grep container
px -L pid
```

![显示线程](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp1zllvtllj31pm0nuag7.jpg)

#### 格式化输出

```shell
ps -eo pid,ppid,pgid,tty,cmd
```

![格式化输出](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp1zoqjad4j31om0tgjwx.jpg)

#### 实时输出

```shell
watch -n 1 'ps -aux --sort -pcpu,-pmem | head -20' 
```

![实时输出](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp1zx2eqygj31x00pi0zs.jpg)

### pstree

pstree命令以树状图显示进程间的关系（display a tree of processes）。ps命令可以显示当前正在运行的那些进程的信息，但是对于它们之间的关系却显示得不够清晰。在Linux系统中，系统调用fork可以创建子进程，通过子shell也可以创建子进程，Linux系统中进程之间的关系天生就是一棵树，树的根就是进程PID为1的init进程。

#### 参数

-p: 显示pid

-a: 显示进程参数

##### 进程树显示进程pid

```shell
pstree -p
```

![pstree -p](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp205oxnd6j30v60u0aii.jpg)

##### 显示指定进程的进程树

```shell
pstree -p 1234
```

![pstree -p](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp206k758dj30us04et9q.jpg)

##### 显示参数

```shell
pstree -ap 1234
```

![pstree -a](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp208117z1j31060amdh6.jpg)

### 参考文档

- https://www.cnblogs.com/huchong/p/10065246.html
- https://zhuanlan.zhihu.com/p/93473461 

