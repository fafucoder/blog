---
title: kubernetes scheme(apimachinery)原理解析
date: 2020-11-29 13:11:23
tags:
- kubernetes
categories:
- kubernetes
---

### 综述

1. K8s 通过scheme注册自定义资源到api-machinery，本质上是空间换时间的一种做法，如果apimachery上没有对应的resourceVersion，直接报错，而不会再去请求k8s-apiserver

2. apimachinery提供了resourceVersion的序列化和反序列化，kubectl 提交的资源类型可以是json格式的，也可以是yaml格式的，在apimachery中会把他解析成对应的struct结构(支持json, yaml, protobuf)
3. apimachery还提供了编码和解码的功能，可以把指定资源的版本进行转化，例如deployment刚开始的版本(忘记是哪个版本了)是v1beta1, 之后是v1版本，通过kubectl convert可以把v1beta1版本转化为v1版本。在apimachinery的内部，都是先把resourceVersion变成internalVersion, 再有internalVersion变成目标版本。(zz_deepcopy就是为了实现Object接口，方便版本转化)

### 参考文档

- https://blog.csdn.net/shida_csdn/article/details/83060586
- https://blog.csdn.net/zhonglinzhang/article/details/105482996
- https://blog.csdn.net/weixin_42663840/article/details/102558455

