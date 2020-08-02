---
title: kubernetes service 原理解析
date: 2020-07-10 15:00:34
tags:
- kubernetes
categories:
- kubernetes
---

### 概念
service 是一组具有相同 label pod 集合的抽象，集群内外的各个服务可以通过 service 进行互相通信，当创建一个 service 对象时也会对应创建一个 endpoint 对象，endpoint 是用来做容器发现的，service 只是将多个 pod 进行关联，实际的路由转发都是由 kubernetes  中的 kube-proxy 组件来实现。

![service原理](https://ask.qcloudimg.com/http-save/474918/c3rxputds9.png?imageView2/2/w/1620)

endpoints controller 是负责生成和维护所有 endpoints 对象的控制器，监听 service 和对应 pod 的变化，更新对应 service 的 endpoints 对象。当用户创建 service 后 endpoints controller 会监听 pod 的状态，当 pod 处于 running 且准备就绪时，endpoints controller 会将 pod ip 记录到 endpoints 对象中，因此，service 的容器发现是通过 endpoints 来实现的。而 kube-proxy 会监听 service 和 endpoints 的更新并调用其代理模块在主机上刷新路由转发规则。

### service 类型
service 默认有四种 ClusterIP、NodePort、LoadBalancer、ExternelName 类型， 此外还有 Ingress

#### ClusterIp
ClusterIP 类型的 service 是 kubernetes 集群默认的服务暴露方式，它只能用于集群内部通信，可以被各 pod 访问，其访问方式为：

```
pod ---> ClusterIP:ServicePort --> (iptables)DNAT --> PodIP:containePort
```

![ClusterIp类型](https://ask.qcloudimg.com/http-save/474918/nlnaj3ikh6.png?imageView2/2/w/1620)


#### nodePort
NodePort 类型的 service 会在集群内部署了 kube-proxy 的节点打开一个指定的端口，之后所有的流量直接发送到这个端口，然后会被转发到 service 后端真实的服务进行访问。Nodeport 构建在 ClusterIP 上，其访问链路如下所示：

`client ---> NodeIP:NodePort ---> ClusterIP:ServicePort ---> (iptables)DNAT ---> PodIP:containePort`

![NodePort类型](https://ask.qcloudimg.com/http-save/474918/qbj7uzictn.png?imageView2/2/w/1620)

#### LoadBalance 
LoadBalancer 类型的 service 通常和云厂商的 LB 结合一起使用，用于将集群内部的服务暴露到外网，云厂商的 LoadBalancer 会给用户分配一个 IP，之后通过该 IP 的流量会转发到你的 service 上。

![locadbanlance类型](https://ask.qcloudimg.com/http-save/474918/l0wvn8je7c.png?imageView2/2/w/1620)

#### ExternelName
通过 CNAME 将 service 与 externalName 的值(比如：foo.bar.example.com)映射起来，这种方式用的比较少。

#### Ingress
Ingress 其实不是 service 的一个类型，但是它可以作用于多个 service，被称为 service 的 service，作为集群内部服务的入口，Ingress 作用在七层，可以根据不同的 url，将请求转发到不同的 service 上。

![ingress](https://ask.qcloudimg.com/http-save/474918/qaq773e8bt.png?imageView2/2/w/1620)


### service服务发现
service 的 endpoints 解决了容器发现问题，但不提前知道 service 的 Cluster IP，怎么发现 service 服务呢？

service 当前支持两种类型的服务发现机制，一种是通过环境变量，另一种是通过 DNS。

#### 环境变量
当一个 pod 创建完成之后，kubelet 会在该 pod 中注册该集群已经创建的所有 service 相关的环境变量，但是需要注意的是，在 service 创建之前的所有 pod 是不会注册该环境变量的，所以在平时使用时，建议通过 DNS 的方式进行 service 之间的服务发现。

#### DNS
可以在集群中部署 CoreDNS 服务(旧版本的 kubernetes 群使用的是 kubeDNS)， 来达到集群内部的 pod 通过DNS 的方式进行集群内部各个服务之间的通讯。

当前 kubernetes 集群默认使用 CoreDNS 作为默认的 DNS 服务，主要原因是 CoreDNS 是基于 Plugin 的方式进行扩展的，简单，灵活，并且不完全被Kubernetes所捆绑。

### 参考文档
- https://segmentfault.com/a/1190000019376912?utm_source=tag-newest
- https://draveness.me/kubernetes-service/
- https://kubernetes.io/zh/docs/tutorials/services/source-ip/
- https://cloud.tencent.com/developer/article/1553954