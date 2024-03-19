---
title: k8s中pod优雅退出
date: 2021-08-11 21:03:13
tags:
- kubernetes
categories:
- kubernetes
---

### 概述

kubernetes提供了一种pod优雅退出机制，使得pod在退出之前可以完成一些资源清理等工作（pod在退出前完成处理正在请求的连接数据等）

### Pod终止流程

1. 当pod被删除时，首先K8S会给旧POD发送SIGTERM信号；将 pod 标记为“Terminating”状态；
2. 于此同时，kube-proxy会更新转发规则，将 Pod 从 service 的 endpoint 列表中摘除掉，新的流量不再转发到该 Pod。
3. 于此同时，如果pod中（的容器）定义了preStop处理程序，kubelet 会调用preStop hook处理程序，假如 preStop hook 的运行时间超出了 grace period（默认30s），kubelet 会发送 SIGTERM 信号并再等 2 秒。
4. `kubelet` 接下来触发容器运行时发送 SIGTERM 信号给每个容器中的init进程。
5. 宽限期结束（grace period）之后，若存在任何一个运行的进程，pod 会收到 SIGKILL 信号。
6. Kubelet 请求 API Server 将此 Pod 资源宽限期设置为0从而完成删除操作。

> 这里注意：当POD被标记为Terminating状态时, preStop和宽限以同步的方式执行；若宽限期结束后，preStop 仍未执行结束，第二步会重新执行并额外获得一个2秒的小宽限期(最后的宽限期)，所以prestop的执行时间注意和terminationGracePeriodSeconds参数配合使用)

### 如何实现pod优雅退出

1. pod可以配置pre_stop钩子（适用于业务进程不处理SIGTERM信号的情况）；
2. pod处理SIGTERM信号；

> 需要注意的是：shell 进程默认不会处理 SIGTERM 信号，自己不会退出，也不会将信号传递给子进程，因此如果容器主进程(init进程)是 shell，业务进程是 shell 的子进程。那么业务进程不会触发停止逻辑。

### 参考文档

- [k8s中pod优雅关闭进程](https://www.cnblogs.com/cuishuai/p/14859182.html)
- [Kubernetes 中如何保证优雅地停止 Pod](https://zhuanlan.zhihu.com/p/59544387)
- [Kubernetes 中 Pod 的优雅退出机制](https://juejin.cn/post/7109292383753699336)
- [Pod 的生命周期](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/pod-lifecycle/)
