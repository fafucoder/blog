---
title: kubernetes kube-proxy实现原理
date: 2020-07-27 11:17:59
tags:
- kubernetes
categories:
- kubernetes
---

### 概念
service 是一组pod的服务抽象，相当于一组pod的LoadBanlance, 负责将请求分发给对应的pod，service会为这个LB提供一个IP，一般称为cluster IP。ClusterIP是个假的IP，这个IP在整个集群中根本不存在，无法通过IP协议栈无法路由，底层underlay设备也无法感知这个IP的存在，因此ClusterIP只能是单主机(Host Only）作用域可见，这个IP在其他节点以及集群外均无法访问。

![service,endpoint和pod关系](https://tva1.sinaimg.cn/large/0081Kckwly1gkbu05yernj313e0nogow.jpg)

kube-proxy的作用主要是负责service的实现，具体来说，就是实现集群内的客户端pod访问service，或者是集群外的主机通过NodePort等方式访问service。

kube-proxy存在于各个node节点上，部署方式的daemonset, 默认使用iptables模式。

![service转发规则](https://tva1.sinaimg.cn/large/0081Kckwly1gkbu0l31npj315q0ocgoq.jpg)

kubernetes中各组件如下：

![kube-proxy](https://tva1.sinaimg.cn/large/0081Kckwly1gkbu12g1x5j30xa0rk0yt.jpg)

### kube proxy模式
kube-proxy 目前有3中常见的proxyMode, 分别是userspace, iptables, ipvs。userspace mode是v1.0及以前版本的默认模式。从v1.1版本开始，增加了iptables mode，在 v1.3版本中正式替代了 userspace 模式成为默认模式（需要 iptables 的版本>= 1.4.11。IPVS 是 LVS 的负载均衡模块，同样基于 netfilter，但比 iptables 性能更好，具备更好的可扩展性。

#### userspace mode
基于用户态的 proxy，service 的请求会先从用户空间进入内核 iptables，然后再回到用户空间，由 kube-proxy 完成后端 endpoints 的选择和代理工作，这种方式流量从用户空间进出内核带来的性能损耗比较大。

![userspace mode](https://tva1.sinaimg.cn/large/0081Kckwly1gkbu4uqamgj31gp0u0b29.jpg)

userspace模式下，kube-proxy 持续监听 Service 以及 Endpoints 对象的变化；对每个 Service，它都为其在本地节点开放一个端口，作为其服务代理端口；发往该端口的请求会采用一定的策略转发给与该服务对应的后端 Pod 实体。kube-proxy 同时会在本地节点设置 iptables 规则，配置一个 Virtual IP，把发往 Virtual IP 的请求重定向到与该 Virtual IP 对应的服务代理端口上。

![userspace请求模式](https://tva1.sinaimg.cn/large/0081Kckwly1gkbu1ouleoj318o0fo45j.jpg)

#### iptables mode
iptables 的方式是完全通过内核的 iptables 实现 service 的代理和 LB。

![iptables mode](https://tva1.sinaimg.cn/large/0081Kckwly1gkbu23nxw5j31140n041r.jpg)

iptables模式相比 userspace 模式，克服了请求在用户态-内核态反复传递的问题，性能上有所提升，但使用 iptables NAT 来完成转发，存在不可忽视的性能损耗

iptables 模式与 userspace 相同，kube-proxy 持续监听 Service 以及 Endpoints 对象的变化；但它并不在本地节点开启反向代理服务，而是把反向代理全部交给 iptables 来实现；即 iptables 直接将对 VIP 的请求转发给后端 Pod，通过 iptables 设置转发策略。

![iptables请求模式](https://tva1.sinaimg.cn/large/0081Kckwly1gkbu2n6mt2j30yu0fq44f.jpg)

#### ipvs mode
ipvs模式是基于 NAT 实现的，通过ipvs的NAT模式，对访问k8s service的请求进行虚IP到POD IP的转发。当创建一个 service 后，kubernetes 会在每个节点上创建一个网卡，同时帮你将 Service IP(VIP) 绑定上。

与iptables、userspace 模式一样，kube-proxy依然监听Service以及Endpoints对象的变化, 不过它并不创建反向代理, 也不创建大量的 iptables 规则, 而是通过netlink 创建ipvs规则，并使用k8s Service与Endpoints信息，对所在节点的ipvs规则进行定期同步; netlink 与 iptables 底层都是基于 netfilter 钩子，但是 netlink 由于采用了 hash table 而且直接工作在内核态，在性能上比 iptables 更优

![ipvs请求模式](https://tva1.sinaimg.cn/large/0081Kckwly1gkbu315s0sj30yy0fk44c.jpg)

### 设置kube-proxy模式
由于kube-proxy是daemonset, 因此通过编辑daemonset发现通过mount configMap把配置传递进去，因此只要修改configmap中的数据就行.

修改kube-proxy configMap中的mode设置kube-proxy模式

```
    kind: KubeProxyConfiguration
    metricsBindAddress: ""
    mode: ""
    nodePortAddresses: null
    oomScoreAdj: null
    portRange: ""
    udpIdleTimeout: 0s
```

### iptables 模式原理解析

iptables 模式下的规则如下所示：

![kube-proxy iptables表](https://docs.cilium.io/en/v1.8/_images/kubernetes_iptables.svg)

#### ClusterIp 模式
1. 非本机pod访问: `PREROUTING --> KUBE-SERVICE --> KUBE-SVC-XXX --> KUBE-SEP-XXX`
2. 本机pod访问: `OUTPUT --> KUBE-SERVICE --> KUBE-SVC-XXX --> KUBE-SEP-XXX`

访问流程：
1. 对于进入 PREROUTING 链的都转到 KUBE-SERVICES 链进行处理；
2. 在 KUBE-SERVICES 链，对于访问 clusterIP 为 10.110.243.155 的转发到 KUBE-SVC-5SB6FTEHND4GTL2W；
3. 访问 KUBE-SVC-5SB6FTEHND4GTL2W 的使用随机数负载均衡，并转发到 KUBE-SEP-CI5ZO3FTK7KBNRMG 和 KUBE-SEP-OVNLTDWFHTHII4SC 上；
4. KUBE-SEP-CI5ZO3FTK7KBNRMG 和 KUBE-SEP-OVNLTDWFHTHII4SC 对应 endpoint 中的 pod 192.168.137.147 和 192.168.98.213，设置 mark 标记，进行 DNAT 并转发到具体的 pod 上，如果某个 service 的 endpoints 中没有 pod，那么针对此 service 的请求将会被 drop 掉；

```
// 1.对于进入 PREROUTING 链的都转到 KUBE-SERVICES 链进行处理；
-A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES

// 2.在 KUBE-SERVICES 链，对于访问 clusterIP 为 10.110.243.155 的转发到 KUBE-SVC-5SB6FTEHND4GTL2W；
-A KUBE-SERVICES -d 10.110.243.155/32 -p tcp -m comment --comment "pks-system/tenant-service: cluster IP" -m tcp --dport 7000 -j KUBE-SVC-5SB6FTEHND4GTL2W

// 3.访问 KUBE-SVC-5SB6FTEHND4GTL2W 的使用随机数负载均衡，并转发到 KUBE-SEP-CI5ZO3FTK7KBNRMG 和 KUBE-SEP-OVNLTDWFHTHII4SC 上；
-A KUBE-SVC-5SB6FTEHND4GTL2W -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-CI5ZO3FTK7KBNRMG
-A KUBE-SVC-5SB6FTEHND4GTL2W -j KUBE-SEP-OVNLTDWFHTHII4SC

// 4.KUBE-SEP-CI5ZO3FTK7KBNRMG 和 KUBE-SEP-OVNLTDWFHTHII4SC 对应 endpoint 中的 pod 192.168.137.147 和 192.168.98.213，
设置 mark 标记，进行 DNAT 并转发到具体的 pod 上，如果某个 service 的 endpoints 中没有 pod，那么针对此 service 的请求将会被 drop 掉；
-A KUBE-SEP-CI5ZO3FTK7KBNRMG -s 192.168.137.147/32 -j KUBE-MARK-MASQ
-A KUBE-SEP-CI5ZO3FTK7KBNRMG -p tcp -m tcp -j DNAT --to-destination 192.168.137.147:7000

-A KUBE-SEP-OVNLTDWFHTHII4SC -s 192.168.98.213/32 -j KUBE-MARK-MASQ
-A KUBE-SEP-OVNLTDWFHTHII4SC -p tcp -m tcp -j DNAT --to-destination 192.168.98.213:7000
```

#### NodePort 模式
在 nodePort 方式下，会用到 KUBE-NODEPORTS 规则链，通过 iptables -t nat -L -n 可以看到 KUBE-NODEPORTS 位于 KUBE-SERVICE 链的最后一个，iptables 在处理报文时会优先处理目的 IP 为clusterIP 的报文，在前面的 KUBE-SVC-XXX 都匹配失败之后再去使用 nodePort 方式进行匹配。

1. 本机模式： `OUTPUT --> KUBE-SERVICE --> KUBE-NODEPORTS --> KUBE-SVC-XXX --> KUBE-SEP-XXX`
2. 非本机模式: `PREROUTING --> KUBE-SERVICE --> KUBE-NODEPORTS --> KUBE-SVC-XXX --> KUBE-SEP-XXX`

访问流程：

该服务的 nodePort 端口为 30070，其 iptables 访问规则和使用 clusterIP 方式访问有点类似，不过 nodePort 方式会比 clusterIP 的方式多走一条链 KUBE-NODEPORTS，其会在 KUBE-NODEPORTS 链设置 mark 标记并转发到 KUBE-SVC-5SB6FTEHND4GTL2W，nodeport 与 clusterIP 访问方式最后都是转发到了 KUBE-SVC-xxx 链。

1. 经过 PREROUTING 转到 KUBE-SERVICES
2. 经过 KUBE-SERVICES 转到 KUBE-NODEPORTS
3. 经过 KUBE-NODEPORTS 转到 KUBE-SVC-5SB6FTEHND4GTL2W
4. 经过KUBE-SVC-5SB6FTEHND4GTL2W 转到 KUBE-SEP-CI5ZO3FTK7KBNRMG 和 KUBE-SEP-VR562QDKF524UNPV
5. 经过 KUBE-SEP-CI5ZO3FTK7KBNRMG 和 KUBE-SEP-VR562QDKF524UNPV 分别转到 192.168.137.147:7000 和 192.168.89.11:7000

```
// 1. 经过 PREROUTING 转到 KUBE-SERVICES
-A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES

// 2. 经过 KUBE-SERVICES 转到 KUBE-NODEPORTS
-A KUBE-SERVICES xxx
-A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS

// 3. 经过 KUBE-NODEPORTS 转到 KUBE-SVC-5SB6FTEHND4GTL2W
-A KUBE-NODEPORTS -p tcp -m comment --comment "pks-system/tenant-service:" -m tcp --dport 30070 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "pks-system/tenant-service:" -m tcp --dport 30070 -j KUBE-SVC-5SB6FTEHND4GTL2W

// 4. 经过KUBE-SVC-5SB6FTEHND4GTL2W 转到 KUBE-SEP-CI5ZO3FTK7KBNRMG 和 KUBE-SEP-VR562QDKF524UNPV
-A KUBE-SVC-5SB6FTEHND4GTL2W -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-CI5ZO3FTK7KBNRMG
-A KUBE-SVC-5SB6FTEHND4GTL2W -j KUBE-SEP-VR562QDKF524UNPV

// 5. 经过 KUBE-SEP-CI5ZO3FTK7KBNRMG 和 KUBE-SEP-VR562QDKF524UNPV 分别转到 192.168.137.147:7000 和 192.168.89.11:7000
-A KUBE-SEP-CI5ZO3FTK7KBNRMG -s 192.168.137.147/32 -j KUBE-MARK-MASQ
-A KUBE-SEP-CI5ZO3FTK7KBNRMG -p tcp -m tcp -j DNAT --to-destination 192.168.137.147:7000
-A KUBE-SEP-VR562QDKF524UNPV -s 192.168.89.11/32 -j KUBE-MARK-MASQ
-A KUBE-SEP-VR562QDKF524UNPV -p tcp -m tcp -j DNAT --to-destination 192.168.89.11:7000
```

### 参考文档
- https://www.cnblogs.com/fuyuteng/p/11598768.html
- https://zhuanlan.zhihu.com/p/94418251?from_voters_page=true
- https://juejin.im/post/6844904098605563912
- https://xuxinkun.github.io/2016/07/22/kubernetes-proxy/
- https://knarfeh.com/2018/07/28/Kubernetes%20%E6%BA%90%E7%A0%81%E7%AC%94%E8%AE%B0%EF%BC%88kube-proxy%EF%BC%89
- https://linuxops.dev/post/kubernetes%E7%9A%84kube-proxy%E7%9A%84%E8%BD%AC%E5%8F%91%E8%A7%84%E5%88%99%E5%88%86%E6%9E%90/