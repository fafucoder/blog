---
title: kubernetes垃圾收集与finalizer
date: 2020-06-04 12:26:14
tags:
- kubernetes
categories:
- kubernetes
---

### 声明式API

声明式表示的是：告诉k8s你要的是什么，而不是告诉他怎么做的命令。

在 K8s 里面，声明式的体现就是 kubectl apply 命令，在对象创建和后续更新中一直使用相同的 apply 命令，告诉 K8s 对象的终态即可，底层是通过执行了一个对原有 API 对象的 PATCH 操作来实现的，可以一次性处理多个写操作，具备 Merge 能力 diff 出最终的 PATCH，而命令式一次只能处理一个写请求。

### finalizer 简介

在一般情况下，如果资源被删除之后，我们虽然能够被触发删除事件，但是这个时候从 Cache 里面无法读取任何被删除对象的信息，这样一来，导致很多垃圾清理工作因为信息不足无法进行，K8s 的 Finalizer 字段用于处理这种情况。在 K8s 中，只要对象 ObjectMeta 里面的 Finalizers 不为空，对该对象的 delete 操作就会转变为 update 操作，具体说就是 update deletionTimestamp 字段，其意义就是告诉 K8s 的 GC“在deletionTimestamp 这个时刻之后，只要 Finalizers 为空，就立马删除掉该对象”。

所以一般的使用姿势就是在创建对象时把 Finalizers 设置好（任意 string），然后处理 DeletionTimestamp 不为空的 update 操作（实际是 delete），根据 Finalizers 的值执行完所有的 pre-delete hook（此时可以在 Cache 里面读取到被删除对象的任何信息）之后将 Finalizers 置为空即可。

### 参考文档

- https://zhuanlan.zhihu.com/p/84051705
- https://zdyxry.github.io/2019/09/13/Kubernetes-%E5%AE%9E%E6%88%98-Operator-Finalizers/?hmsr=codercto.com&utm_medium=codercto.com&utm_source=codercto.com

- https://kubernetes.io/zh/docs/concepts/workloads/controllers/garbage-collection/
- https://draveness.me/kubernetes-garbage-collector/