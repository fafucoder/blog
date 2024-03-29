---
title: linux 终端控制
date: 2021-03-30 11:22:59
tags:
- linux
categories:
- linux
---

### 问题

1. linux打开一个终端运行一个程序，在程序运行未结束的时候如果关掉终端的话，那么该程序也会跟着退出。
2. 如果终端运行的程序堵塞了，则整个终端窗口将无法再执行其他程序。

![进程](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp243alnjxj31e008q111.jpg)

为了使进程不堵塞整个终端，可使用如下命令使进程在后台运行:

```shell
nohup <command> &
```

![nohup](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp24hwww9wj30zm01sdgi.jpg)

其中： 

- nohup表示让我们的程序进程忽略所有挂断（SIGHUP）信号。
- “&”表示让我们的程序进入后台运行

### 基础知识

##### 进程

进程是linux进行分配资源的最小单位，分为前台进程和后台进程

- 前台进程：例如这样： hexo server
- 后台进程： 例如这样： hexo sever &

##### 进程组

每个进程除了有一进程ID外，还属于一个进程组，进程组就是一个或多个进程的集合。进程组中有一个进程组长，组长的进程 ID 是进程组 ID(PGID)。

当我们在终端中敲下一条命令，然后按下回车的时候，Shell会开启一个新进程来执行这条命令。如果这条命令是由管道连接起来的多个命令组成的话，Shell便会开启多个进程来执行这一组任务。无论是单独的一条命令，还是由管道连接的多条命令，都会被放入到一个新的**进程组(任务)**中。只包含一条命令的时候，就会创建一个由一个进程组成的进程组。进程组中的每个进程都具有相同的进程组标识符(进程组ID)，这个进程组标识符其实就是进程组中某个进程(即进程组组长)的进程ID。**一个进程组也被称为「作业」**

![进程组](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp25p3hlajj30wy04yt9d.jpg)

如上截图所示：

- bash：进程和进程组ID都是2571(父进程是sshd进程)
- ps: 进程和进程组ID都是14245，父进程是bash(2571),因为是在shell上执行的命令
- cat: 进程组ID和ps的进程组相同，父进程为bash(2571)进程

那为啥Linux里要有进程组呢？其实，提供进程组就是为了方便对进程进行管理。假设要完成一个任务，需要同时并发100个进程。当用户处于某种原因要终止 这个任务时，要是没有进程组，就需要手动的一个个去杀死这100个进程。

##### 守护进程

守护进程是一种长期运行的后台服务进程，通常以daemon的方式运行，通常以字母d结尾，与其他进程相比较，它大概有如下特点：

- 无需控制终端(不需要与用户交互)
- 在后台运行
- 生命周期比较长，一般随系统启动和关闭

##### 会话

**由一个或者多个进程组组成的集合，**一个会话中的所有进程都具有相同的会话标识符，会话首进程是指创建会话的进程，其进程ID会成为会话ID。

**使用会话最多的是支持任务控制的shell,由shell创建的所有进程组及shell自身隶属与同一个会话，shell自身就是该会话的会话首进程**。

![session id](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp269e4zjcj30ya052t9h.jpg)

在任意时刻，会话中总有一个前台进程组(前台任务)，可以从终端读取输入，向终端发送输出。如果用户输入Ctrl+C或者Ctrl+Z,就可以分别让任务终止或挂起。同时，一个会话还可以拥有任意多个后台进程组，后台进程组可以用’&’结尾的命令行创建。

![会话](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp263d7z3bj31v00nqgsy.jpg)

 如图，该会话中有三个进程组。通常是由shell的管道将几个进程编成一组的。上图有可能是由下列形式的shell命令形成的:

```shell
$ proc1 | proc2 & 
$ proc3 | proc4 | proc5
```

##### 终端

终端(Terminal)也是一台物理设备，只用于输入输出，本身没有强大的计算能力。一台计算机只有一个控制台，在计算资源紧张的时代，人们想共享一台计算机，可以通过终端连接到计算机上，将指令输入终端，终端传送给计算机，计算机完成指令后，将输出传送给终端，终端将结果显示给用户。

##### 登录终端

在早期的计算机上面，用户用哑终端（用硬连接连到主机）进行登录，这种登录要经由内核的终端设备驱动程序。因为连到主机上的终端设备数是固定的，所以同时的登录数也就有了已知的上限。随着图形终端的出现，创建终端窗口的应用也被开发出来，它仿真了基于字符的终端，使用户可以用熟悉的方式（shell命令行）与主机进行交互。包括使用网络进行远程登录的远程终端也是使用的这种方式。

##### 伪终端

随着图形终端的出现，创建终端窗口的应用也被开发出来，它仿真了基于字符的终端，使用户可以用熟悉的方式（shell命令行）与主机进行交互。包括使用网络进行远程登录的远程终端也是使用的这种方式。网络登录与传统的串行终端登录的区别在于，前者必须等待一个网络连接请求到达，而不是使一个进程等待每一个可能的登录。为了使同一个软件既能处理终端登录又能处理网络登录，系统使用了一种称为伪终端（pseudo terminal）的软件驱动程序，
它仿真串行终端的运行行为，并将终端操作映射为网络操作。

##### 控制终端

- 当一个终端与一个会话相关联后，那么这个终端就称为该会话的控制终端(controlling terminal)。
- 建立与控制终端连接的会话首进程被称为控制进程(controlling process)。
- 一个会话中的几个进程组可被分成一个前台进程组(foreground process group)以及一个或多个后台进程组(background process group)。
- 如果一个会话有一个控制终端的话， 则它有一个前台进程组，其他进程组为后台进程组。
- 无论何时键入终端的中断键或退出键，都会将中断信号或退出信号发送至前台进程组的所有进程。
- 如果终端检测到调制解调器（或网络）断开，则挂断信号（SIGHUP）发送至控制进程（会话首进程），如果**会话首进程退出,则将挂断信号（SIGHUP）发送至前台进程组的所有进程**。

![控制终端](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp26l1y3u1j31gn0u0tmu.jpg)

### 相关命令

#### jobs

查看当前所有的后台进程

![jobs](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp272ozs1uj30sa038wet.jpg)

#### fg

将后台运行的命令调至前台继续运行, fg 命令格式为`fg %作业号` 其中 `％ `可以省略，但若将`% 工作号`全部省略，则此命令会将带有 + 号的工作恢复到前台。另外，使用此命令的过程中， % 可有可无。

![fg](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp277qt4mfj313m0n2tcf.jpg)

##### bg

将一个在后台暂停的命令变成继续执行状态

![bg](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp279l6n8qj30w809w75s.jpg)

##### ctrl + z
可以将一个正在前台执行的命令放到后台，并且暂停

#### tty

tty命令看看当前bash关联到了哪个tty

![tty](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp27c81h2mj30sq022q30.jpg)

##### lsof

lsof 用于查看你进程打开的文件，打开文件的相关进程，进程打开的端口(TCP、UDP)等，在linux环境下，任何事物都以文件的形式存在，通过文件不仅仅可以访问常规数据，还可以访问网络连接和硬件。

![lsof](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp27f1dw3gj310q08o76r.jpg)

###### 参数

| 参数       | 描述                                             |
| :--------- | :----------------------------------------------- |
| -a         | 列出打开文件存在的进程                           |
| -c<进程名> | 列出指定进程所打开的文件                         |
| -g         | 列出GID号进程详情                                |
| -d<文件号> | 列出占用该文件号的进程                           |
| +d<目录>   | 列出目录下被打开的文件                           |
| +D<目录>   | 递归列出目录下被打开的文件                       |
| -n<目录>   | 列出使用NFS的文件                                |
| -i<条件>   | 列出符合条件的进程。（4、6、协议、:端口、 @ip ） |
| -p<进程号> | 列出指定进程号所打开的文件                       |
| -u         | 列出UID号进程详情                                |
| -h         | 显示帮助信息                                     |
| -v         | 显示版本信息                                     |

###### 查看哪个进程正在使用某个文件

```
lsof /bin/bash
```

![lsof /bin/bash](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp27n2nuv8j30wq054jsv.jpg)

###### 查看哪个进程打开指定端口

```
lsof -i :22
```

![losf -i :22](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp27p127ffj319i06kdig.jpg)

###### 查看指定进程打开的文件信息

```
lsof -p 1257
```

![lsof -p 1257](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp27r9r1wej30uj0u07s9.jpg)

### 参考文档

- https://www.linuxidc.com/Linux/2019-02/157126.htm
- https://segmentfault.com/a/1190000009082089
- https://segmentfault.com/a/1190000009152815
- http://shareinto.k8s.101.com/2016/11/17/linux-terminal/
- https://blog.csdn.net/caoshangpa/article/details/80140888
- https://andrewpqc.github.io/2018/10/31/linux-process-group-and-session-and-jobs-manage/
- https://www.cnblogs.com/huchong/p/10095152.html

