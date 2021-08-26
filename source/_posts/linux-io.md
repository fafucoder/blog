---
title: linux io多路复用
date: 2020-09-14 20:22:13
tags:
- linux
categories:
- linux
---

### socket 概念
socket这个词可以表示很多概念，在TCP/IP协议中“IP地址 + TCP或UDP端口号”唯一标识网络通讯中的一个进程，“IP + 端口号”就称为socket。在TCP协议中，建立连接的两个进程各自有一个socket来标识，那么两个socket组成的socket pair就唯一标识一个连接。

### 进程堵塞 概念

### 多路复用解决的问题

### 参考文档
- https://wumingx.com/performance/io-epoll.html
- https://xie.infoq.cn/article/5241a48ffdb62ec3ba0235733
- https://blog.csdn.net/jyy305/article/details/73012706
- https://colobu.com/2019/02/23/1m-go-tcp-connection/