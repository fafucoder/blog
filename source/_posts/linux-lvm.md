---
title: 使用LVM实现linux扩容
date: 2021-08-15 22:06:47
tags:
- linux
categories:
- linux
---

### 概述

​	玩过虚拟机的同学肯定遇到过磁盘容量100%而无法安装新的软件这一问题(因为磁盘分配不合理或者磁盘挂载的盘大小太小了导致的)， 磁盘空间不足的通常方法就是挂在一块新的盘到虚拟机，然后把占用磁盘空间多的目录挂载到这块新的盘上去(我之前就都是这么干的^_^), 或者使用LVM来实现磁盘的扩容(使用LVM扩容才是正解 _^_  ), 下面就来了解下LVM的知识，已经使用lvm如何实现磁盘扩容吧。

### 参考文档

- [使用LVM](https://www.junmajinlong.com/linux/lvm/)   //骏马金龙老师的文章，推荐阅读
- https://www.cnblogs.com/tiantianhappy/p/10143663.html centos扩容
- https://aurthurxlc.github.io/Aurthur-2017/Centos-7-extend-lvm-volume.html   扩容