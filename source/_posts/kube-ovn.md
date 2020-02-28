---
title: kube-ovn学习指南
date: 2020-01-09 22:31:08
tags: 
- kubernetes
- kube-ovn
---

### kube-ovn的系统架构
![kube-ovn系统架构](http://dockone.io/uploads/article/20190710/decaf6ff139442ffeccd70b7ad3d0bc3.png)

Kube-OVN 自身的逻辑主要集中在图中蓝色的部分 kube-ovn-controller，kube-ovn-cni 和 kube-ovn-cniserver。

kube-ovn-controller 可以看成是一个 Kubernetes 资源的控制器，它会 watch Kubernetes 内所有和网络相关的资源变化，例如 Pod、Node、Namespace、Service、Endpoint 和 NetworkPolicy。每当资源发生变化 kube-ovn-controller 会计算预期的状态，并将网络对应的变化翻译成 OVN 北向数据库的资源对象。同时 kube-ovn-controller 会将配置具体网络的信息，例如分配的 IP、Mac、网关等信息再以 Annotation 的方式回写到 Kubernetes 的资源里，方便后续的处理。

kube-ovn-cni主要工作是适配 CNI 的协议和 Kubelet 对接，将标准的 cni add/del 命令发送到每台机器上的 kube-ovn-cniserver 进行后续处理。

kube-ovn-cniserver 会根据传入的参数反查 Kubernetes 中资源对应的 Annotation，操作每台机器上的 OVS 以及容器网络。

![kube-ovn流程图](ylt.jpg)

kubeovn-controller watch到数据后操作的是ovn, 而cni-server操作的是ovs, 这两个组件之间是通过annotation进行关联通信的。

以创建 Pod 的例子，Pod 下发到 apiserver 后 kube-ovn-controller 会 watch 的新生成了一个 Pod，然后调用 ovn-nb 的接口去创建一个虚拟交换机接口，成功后将 OVN 分配的 IP、Mac、GW 等信息反写到这个 Pod 的 Annotation 中。接下来 kubelet 创建 Pod 时会调用 kube-ovn-cni，kube-ovn-cni 将信息传递给 kube-ovn-cniserver。CNIServer 会反查 Pod 的 Annotation 获得具体的 IP 和 Mac 信息来配置本地的 OVS 和容器网卡，完成整个工作流。其他的 Service、NetworkPolicy 的流程也是和这个类似。

### kube-ovn的网络模型
![kube-ovn网络模型](http://dockone.io/uploads/article/20190710/b423db902176e17b0108ffb62f69cf38.jpeg)

Kube-OVN 采用了一个 Namespace 一个子网的模型，子网是可以跨节点的这样比较符合用户的预期和管理。每个子网对应着 OVN 里的一个虚拟交换机，LB、DNS 和 ACL 等流量规则现在也是应用在虚拟交换机上，这样方便之后做更细粒度的权限控制，例如实现 VPC，多租户这样的功能。

所有的虚拟交换机目前会接在一个全局的虚拟路由器上(ovn-vsctl)，这样可以保证默认的容器网络互通，未来要做隔离也可以很方便的在路由层面进行控制。此外还有一个特殊的 Node 子网，会在每个宿主机上添加一块 OVS 的网卡，这个网络主要是把 Node 接入容器网络，使得主机和容器之间网络可以互通。需要注意的是这里的虚拟交换机和虚拟路由器都是逻辑上的，实现中是通过流表分布在所有的节点上的，因此并不存在单点的问题。

网关负责访问集群外部的网络，目前有两种实现，一种是分布式的，每台主机都可以作为运行在自己上面的 pod 的出网节点。另一种是集中式的，可以一个 Namespace 配置一个网关节点，作为当前 Namespace 里的 Pod 出网所使用的网关，这种方式所有出网流量用的都是特定的 IP nat 出去的，方便外部的审计和防火墙控制。当然网关节点也可以不做 nat 这样就可以把容器 IP 直接暴露给外网，达到内外网络的直连。

### kube-ovn源代码

#### kube-ovn-controller
![kube-ovn-controller运行流程图](kube-ovn-controller.png)

#### kube-ovn-cniserver
![kube-ovn-server运行流程图](kube-ovn-cniserver.png)

### 参考网址
- http://weekly.dockerone.com/article/9076  //关于kube-ovn的网络架构和工作原理
- https://github.com/alauda/kube-ovn/wiki   //kube-ovn如何使用
- https://neuvector.com/network-security/advanced-kubernetes-networking/  //其他网络插件流程
- https://www.jianshu.com/p/a58514b34f54 //从cni到ovn网络