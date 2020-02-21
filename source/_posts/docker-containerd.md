---
title: docker容器运行时containerd
date: 2020-01-31 15:51:26
tags:
- docker
- containerd
---

### container概念

一句话解释，就是`一组受到资源限制，彼此间相互隔离的进程`。实现起来也并不复杂，隔离所用到的技术都是由linux内核本身提供的（所以说目前绝大部分的容器都是必须要跑在linux里面的）。其中namespace用来做访问隔离（每个容器进程都有自己独立的进程空间，看不到其他进程），cgroups用来做资源限制（cpu、内存、存储、网络的使用限制）。总的来说容器就是一种基于操作系统能力的隔离技术，这和基于hypervisor的虚拟化技术（能完整模拟出虚拟硬件和客户机操作系统）复杂度不可同日而语。

![docker container](https://upload-images.jianshu.io/upload_images/14871146-b3bbac53379514ac.png)

可以看出，容器是没有自己的OS的，直接共享宿主机的内核，也没有hypervisor这一层进行资源隔离和限制，所有对于容器进程的限制都是基于操作系统本身的能力来进行的，由此容器获得了一个很大的优势：轻量化，由于没有hypervisor这一层，也没有自己的操作系统，自然占用资源很小，而打出来的镜像文件也要比虚拟机小的多。

### containerd概念

从Docker 1.11开始，Docker容器运行已经不是简单的通过Docker daemon来启动，而是集成了containerd、runC等多个组件， 其中，containerd独立负责容器运行时和生命周期（如创建、启动、停止、中止、信号处理、删除等），其他一些如镜像构建、卷管理、日志等由Docker Daemon的其他模块处理。

containerd是容器技术标准化之后的产物，为了能够兼容 OCI 标准，将容器运行时及其管理功能从 Docker Daemon 剥离。从理论上来说，即使不运行 dockerd，也能够直接通过 containerd 来管理容器。

containerd向上为Docker Daemon提供了gRPC接口，使得Docker Daemon屏蔽下面的结构变化。向下通过containerd-shim结合runC，使得引擎可以独立升级，避免之前Docker Daemon升级会导致所有容器不可用的问题。

containerd-shim称之为垫片，每启动一个容器都会起一个新的docker-shim的一个进程， 他直接通过指定的三个参数：容器id，boundle目录， 运行是二进制（默认为runc）来调用runc的api创建一个容器（比如创建容器：最后拼装的命令如下：runc create 。。。。。）

runC是从Docker的libcontainer中迁移而来的，实现了容器启停、资源隔离等功能

![containerd原理图](https://jiajunhuang.com/articles/img/containerd_architecture.png)

### 总结
分离后的docker运行原理如下所示：

![docker结构图](https://upload-images.jianshu.io/upload_images/8911567-a2909ee9253d3e1a.png)

### 参考手册
- https://www.jianshu.com/p/52c0f12b0294        //docker，containerd，runc，docker-shim
- https://www.infoq.cn/article/2017/01/Docker-Containerd-OCI-1   //Docker 开源容器运行时组件 Containerd
- https://www.jianshu.com/p/517e757d6d17        //深入理解容器基础概念