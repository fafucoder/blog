---
title: linux中stdin, stdout, stderr
date: 2021-03-07 11:45:51
tags:
- linux
categories:
- linux
---

### 文件描述符
对Linux进程来讲，每个打开的文件都是通过文件描述符(File Descriptor)来标识的，内核为每个进程维护了一个文件描述符表，这个表以FD为索引，再进一步指向文件的详细信息。在进程创建时，内核为进程默认创建了0、1、2三个特殊的FD，这就是STDIN、STDOUT和STDERR。
![文件描述符](https://tva1.sinaimg.cn/large/008eGmZEly1gobnjioex4j318c0iyjwp.jpg)

I/O重定向也就是让已创建的FD指向其他文件。比如，下面是对STDOUT重定向到testfile.txt前后内核文件描述符表变化的示意图
![io重定向](https://tva1.sinaimg.cn/large/008eGmZEly1gobnkjht9yj31qe0q6n0f.jpg)

在I/O重定向的过程中，不变的是FD 0/1/2代表STDIN/STDOUT/STDERR，变化的是文件描述符表中FD 0/1/2对应的具体文件，  Shell正是通过I/O重定向和管道这种特殊的文件把多个程序的STDIN和STDOUT串联在一起组成更复杂功能的，下面是Shell中通过管道的示意图：
![shell 管道](https://tva1.sinaimg.cn/large/008eGmZEly1gobnmgdm7qj31jy0g0aci.jpg)

**示例：**
![proc](https://tva1.sinaimg.cn/large/008eGmZEly1gobnofym2sj316m0kaaf4.jpg)


#### pgrep 命令
pgrep查看当前正在运行的进程，并将与选择条件匹配的进程ID列出到stdout（屏幕）。当你想要某个进程的PID时，pgrep很方便。(通过`ps -ef | grep xxx`也能实现)
![pgrep](https://tva1.sinaimg.cn/large/008eGmZEly1gobo1kab79j31gg04wabi.jpg)

选项#
- -o：仅显示找到的最小（起始）进程号；
- -n：仅显示找到的最大（结束）进程号；
- -l：显示进程名称；
- -P：指定父进程号；
- -g：指定进程组；
- -t：指定开启进程的终端；
- -u：指定进程的有效用户ID。

### 参考文档
- https://www.cnblogs.com/weidagang2046/p/io-redirection.html
- https://zhuanlan.zhihu.com/p/99120125