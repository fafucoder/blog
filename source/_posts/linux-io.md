---
title: linux I/O多路复用技术
date: 2020-09-14 20:22:13
tags:
- linux
categories:
- linux
---

### 什么是文件描述符FD

> 阅读文件描述符之前，推荐先看 [之前进程，线程，协程的文章](https://fafucoder.github.io/2021/03/06/linux-process/)

Linux 系统中，把一切都看做是文件，当进程打开现有文件或创建新文件时，内核向进程返回一个文件描述符，文件描述符就是内核为了高效管理已被打开的文件所创建的索引，用来指向被打开的文件，所有执行I/O操作的系统调用都会通过文件描述符。文件描述符在形式上是一个非负整数。实际上，它是一个索引值，指向内核为每一个进程所维护的该进程打开文件的记录表。

在linux中，用task_struct 表示一个进程：

```c
struct task_struct {
   long               state;     // 虚拟内存结构体

   struct mm_struct   *mm;       // 进程内存管理信息

   pid_t              pid;       // 进程号

   struct task_struct *parent;   // 指向父进程的指针

   struct list_head children;    // 子进程列表

   struct fs_struct* fs;         // 存放文件系统信息的指针

   struct files_struct *files;   // 一个数组，包含该进程打开的文件指针
}
```

一个task_struct包含很多选项，其中包括两个选项:

- mm：指向的是进程的虚拟内存，也就是载入资源和可执行文件的地方；
- files：指针指向一个数组，这个数组里装着所有该进程打开的文件的指针。

#### 文件描述符与文件的详细关系

files是一个文件指针数组。一般来说，一个进程会从files[0]读取输入，将输出写入files[1]，将错误信息写入files[2]。

每个进程被创建时，files的前三位被填入默认值，分别指向标准输入流、标准输出流、标准错误流。我们常说的「文件描述符」就是指这个文件指针数组的索引 ，所以程序的文件描述符默认情况下 0 是输入，1 是输出，2 是错误。( 此处你应该查看[这篇文章](https://fafucoder.github.io/2021/03/07/linux-std/) )

![fd](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gwp4eurb76j319y0rwmzj.jpg)

如果我们写的程序需要其他资源，比如打开一个文件进行读写，这也很简单，进行系统调用，让内核把文件打开，这个文件就会被放到files的第 4 个位置，对应文件描述符 3：

![文件描述符3](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gwp4dxupoej30u00gat9y.jpg)

在操作系统中，为了维护文件描述符，建立了三个表(三个数据结构), 分别为`进程级的文件描述符表`, `系统级的文件描述符表`, `文件系统的i-node表`，不同表的记录内容如下所示：

![文件描述符](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gwp4phqrxcj30xu0gygnx.jpg)

下图展示的是文件描述符与打开的文件和i-node之间的关系:

![i-node关系](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gwp4rcsj1bj314y0lg40p.jpg)

1. 在进程A中，文件描述符 fd 1和fd 30都指向了同一个打开的文件句柄（#23），这可能是通过调用dup()、dup2()、fcntl()或者对同一个文件多次调用了open()函数而形成的。

2. 进程A中的文件描述符fd 2和进程B的文件描述符fd 2都指向了同一个打开的文件句柄（#73），这种情况有几种可能：

   1): 进程A和进程B可能是父子进程关系(调用 fork() 后出现);

   2): 进程A和进程B都调用open函数打开了同一个文件，此时描述符恰好一致 (低概率事件);

   3): 进程A和进程B中某个进程通过UNIX域套接字将一个打开的文件描述符传递给另一个进程。

3. 进程A的描述符fd 0和进程B的描述符fd 3分别指向不同的打开文件句柄，但这些句柄均指向i-node表的相同条目（#1936）换言之，指向同一个文件。发生这种情况是因为每个进程各自对同一个文件发起了打开请求。（同一个进程两次打开同一个文件，也会发生类似情况。）

#### 文件描述符的限制

在编写文件操作的或者网络通信的软件时，初学者一般可能会遇到“Too many open files”的问题。这主要是因为文件描述符是系统的一个重要资源，虽然说系统内存有多少就可以打开多少的文件描述符，但是在实际实现过程中内核是会做相应的处理的，一般最大打开文件数会是系统内存的10%(称之为系统级限制)，与此同时，内核为了不让某一个进程消耗掉所有的文件资源，其也会对单个进程最大打开文件数做默认值处理（称之为用户级限制），默认值一般是1024。

> 查看系统级别的最大打开文件数可以使用`sysctl -a | grep fs.file-max`命令查看；
> 查看用户级别的最大打开文件数可以使用使用 `ulimit -n `命令查看；

### socket 概念

socket这个词可以表示很多概念，在TCP/IP协议中“IP地址 + TCP或UDP端口号”唯一标识网络通讯中的一个进程，“IP + 端口号”就称为socket。在TCP协议中，建立连接的两个进程各自有一个socket来标识，那么两个socket组成的socket pair就唯一标识一个连接。

在Unix/Linux系统下，一个socket句柄，可以看做是一个文件，在socket上收发数据，相当于对一个文件进行读写，所以一个socket句柄，通常也用文件句柄的fd来表示。在linux中，一个socket fd 可能如下表示：

```livescript
root@ubuntu:~# ll /proc/1583/fd  
total 0  
lrwx------ 1 root root 64 Jul 19 12:37 7 -> socket:[18892]  
lrwx------ 1 root root 64 Jul 19 12:37 8 -> socket:[18893] 
```

这里我们看到 fd 7、8 都是一个 socket fd，名字为 socket:[18892]、socket:[18893], 名字中的socket 标识这是一个 socket 类型的 fd， 数字[18892] 表示这个是一个 inode 号，能够唯一标识本机的一条网络连接。

套接字由 socket() 系统调用创建，但是套接字其实可分为两种类型，监听套接字和普通套接字。

对于监听套接字，不走数据流，只管理连接的建立。`accept` 将从全连接队列获取一个创建好的 socket（ 3 次握手完成），对于监听套接字的可读事件就是全连接队列非空。对于监听套接字，我们只在乎可读事件。

普通套接字就是走数据流的，也就是网络 IO，针对普通套接字我们关注可读可写事件。

监听套接字是由 listen() 把 socket fd 转化而成。listenfd称为监听套接字（listening socket）描述符，它的生命周期与服务器的生命周期一致且一般一个服务器保有一个listenfd。connfd是已连接套接字（connected socket）描述符，一个连接对应一个connfd，当连接关闭时connfd也会被关闭。值得一提的是connfd是在完成TCP三次握手后被创建。(总结起来就是listenfd是内核维护的，connfd是应用程序自己维护的)

![socket](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gwp608bb6pj30u00uh0ub.jpg)

### 全连接和半连接队列

上面提到过全连接队列和半连接队列，什么是全连接队列，什么是半连接队列呢，在说这个概念之前，我们先说TCP的三次握手如下：

1. client 发送 SYN 到server 发起握手；
2. server 收到 SYN后回复SYN+ACK给client；
3. client 收到SYN+ACK后，回复server一个ACK表示收到了server的SYN+ACK，这时表示连接建立完成。

![TCP三次握手](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/202401172017128.png)

全连接队列和半连接队列其实就是三次握手中连接的状态流转半连接队列，也称 SYN 队列；全连接队列，也称 accept 队列）

服务端收到客户端发起的 SYN 请求后，**内核会把该连接存储到半连接队列**，并向客户端响应 SYN+ACK，接着客户端会返回 ACK，服务端收到第三次握手的 ACK 后，**内核会把连接从半连接队列移除，然后创建新的完全的连接，并将其添加到 accept 队列，等待进程调用 accept 函数时把连接取出来。**

![全连接队列与半连接队列](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/202401172045208.png)

### 进程堵塞概念

### 多路复用解决的问题

### 参考文档
- https://wumingx.com/performance/io-epoll.html
- https://xie.infoq.cn/article/5241a48ffdb62ec3ba0235733
- [百万 Go TCP 连接的思考: epoll方式减少资源占用](https://colobu.com/2019/02/23/1m-go-tcp-connection/)  // 鸟窝大神
- [I/O多路复用](https://nxw.name/2021/i-oduo-lu-fu-yong-f6ebd183)   // 这老哥的文章不错哈，值得阅读
- [文件描述符（File Descriptor）简介](https://segmentfault.com/a/1190000009724931)  // 文件描述符的相关知识， 图画的不错~
- [Linux fd 系列 — socket fd 是什么？](https://mp.weixin.qq.com/s/-Cntd83HR1TcSUvmoT5H7A) // 这篇文章写的属实不错，学习了~

