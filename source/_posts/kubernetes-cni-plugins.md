---
title: kubernetes常见的网络插件
date: 2021-01-03 14:02:52
tags:
- kubernetes
categories:
- kubernetes
---

### 概述

常见的容器网络方案可以从协议栈层级、穿越形态、隔离方式这三种形式进行划分。

**协议栈层级:**

- 第一种：协议栈二层。
- 第二种：协议栈三层（纯路由转发）。
- 第三种：协议栈二层加三层。

**穿越形态：**

按穿越的形态划分，这个与实际部署环境十分相关。穿越形态分为两种：Underlay、Overlay。

- Underlay：在一个较好的一个可控的网络场景下，我们一般利用 Underlay。可以这样通俗的理解，无论下面是裸机还是虚拟机，只要网络可控，整个容器的网络便可直接穿过去 ，这就是 Underlay。
- Overlay：Overlay 在云化场景比较常见。Overlay 下面是受控的 VPC 网络，当出现不属于 VPC 管辖范围中的 IP 或者 MAC，VPC 将不允许此 IP/MAC 穿越。出现这种情况时，我们都利用 Overlay 方式来做。

**隔离方式：**

隔离方式与多种户型相关（用户与用户之间的隔离方式），隔离方式分为 FLAT、VLAN、VXLAN 三种：

- FLAT：纯扁平网络，无隔离；
- VLAN：VLAN 机房中使用偏多，但实际上存在一个问题？就是它总的租户数量受限。众所周知，VLAN 具有数量限制。
- VXLAN：VXLAN 是现下较为主流的一种隔离方式。因为它的规模性较好较大，且它基于 IP 穿越方式较好。

### fannel

Kubernetes中解决网络跨主机通信的一个经典插件就是Flannel。flannel因为其简单的特性而广为熟知，在现在的flannel实现方案中，有三种方式，分别为udp模式，host-gw模式，vxlan模式

#### udp模式

UDP是最早的实现方式，但是由于其性能原因，现已经被废弃，但是UDP模式是最直接，也最容易理解的跨主机实现方式。

假如有两台Node，如下：

1. Node01上有容器nginx01，其IP为172.20.1.107，其docker0的地址为172.20.1.1/24；
2. Node02上有容器nginx02，其IP为172.20.2.133，其docker0的地址为172.20.2.1/24；

那么现在nginx01要访问nginx02，其流程应该是怎么样的呢？

1. 首先从nginx01发送IP包，源IP是172.20.1.107，目的IP是172.20.2.133。
2. 由于目的IP并不在Node01上的docker0网桥里，所以会将包通过默认路由转发到docker0网桥所在的[宿主机](https://cloud.tencent.com/product/cdh?from=10680)上；
3. 它会通过本地的路由规则，转发到下一个目的IP，我们可以通过ip route查看本地的路由信息，通过路由信息可以看到它被转发到一个flannel0的设备中；
4. flannel0设备会把这个IP包交给创建这个设备的应用程序，也就是Flannel进程（从内核状态向用户状态切换）；
5. Flannel进程收到IP包后，将这个包封装在UDP中，就根据其目的地址将其转发给Node02（通过每个宿主机上监听的8285端口），这时候的源地址是Node01的地址，目的地址是Node02的地址；
6. Node02收到包后，就会直接将其转发给flannel0设备，然后进行解包，匹配本地路由规则转发给docker0网桥，然后docker0网桥就扮演二层交换机的功能，将包转发到最终的目的地；

![udp模式](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gw23bwzmruj31i70u0tc1.jpg)

> 注：
>  1、flannel0是一个TUN设备，它的作用是在操作系统和应用程序之间传递IP包；
>  2、Flannel是根据子网（Subnet）来查看IP地址对应的容器是运行在那个Node上；
>  3、这些子网和Node的对应关系，是保存在Etcd中（仅限UDP模式）； 
>  4、UDP模式其实是一个三层的Overlay网络；它首先对发出的IP包进行UDP封装，然后接收端对包进行解封拿到原始IP，进而把这个包转发给目标容器。这就好比Flannel在不同的宿主机上的两容器之间打通了一条隧道，使得这个两个IP可以通信，而无需关心容器和宿主机的分布情况；

UDP之所以被废弃是主要是由于其仅在发包的过程中就在用户态和内核态进行来回的数据交换，这样的性能代价是很高的.

![udp模式缺陷](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gw23h7ejwij31910u0wg3.jpg)

#### vxlan 模式

VXLAN：Virtual Extensible LAN（虚拟可扩展局域网），是Linux内核本身就支持的一种虚拟化网络技术，它可以完全在内核态实现上述的封装和解封装过程，减少用户态到内核态的切换次数，把核心的处理逻辑都放到内核态，其通过与前面相似的隧道技术，构建出覆盖网络或者叠加网络（Overlay Network）。

其设计思想为在现有的三层网络下，叠加一层虚拟的并由内核VXLAN维护的二层网络，使得连接在这个二层网络上的主机可以像在局域网一样通信。

为了能够在二层网络中打通隧道，VXLAN会在宿主机上设置一个特殊的网络设备作为隧道的两端，这个隧道就叫VTEP（Virtual Tunnel End Point 虚拟隧道端点）。而VTEP的作用跟上面的flanneld进程非常相似，只不过它进行封装和解封的对象是二层的数据帧，而且这个工作的执行流程全部在内核中完成。

![](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gw23vi1k3jj314q0mcabk.jpg)

我们可以看到每台Node上都有一个flannel.1的网卡，它就是VXLAN所需要的VTEP设备，它既有IP地址，也有MAC地址。 现在我们nginx01要访问nginx02，其流程如下：

1. nginx01发送请求包会被转发到docker0；
2. 然后会通过路由转发到本机的flannel,1；
3. flannel.1收到包后通过ARP记录找到目的MAC地址，并将其加原始包上，封装成二层数据帧（将源MAC地址和目的MAC地址封装在它们对应的IP头外部）；
4. Linux内核把这个数据帧封装成普通的可传输的数据帧，通过宿主机的eth0进行传输（也就是在原有的数据帧上面加一个VXLAN头VNI，它是识别某个数据帧是不是归自己处理的的重要标识，而在flannel中，VNI的默认值就是1，这是由于宿主机上的VTEP设备名称叫flannel.1，这里的1就是VNI的值）；
5. 然后Linux内核会把这个数据帧封装到UDP包里发出去；
6. Node02收到包后发现VNI为 1，Linux内核会对其进行解包，拿到里面的数据帧，然后根据VNI的值把它交给Node02上的flannel.1设备，然后继续进行接下来的处理；

在这种场景下，flannel.1设备实际扮演的是一个网桥的角色，在二层网络进行UDP包的转发，在Linux内核中，网桥设备进行转发的依据是一个叫做FDB（Foewarding Database）的转发[数据库](https://cloud.tencent.com/solution/database?from=10680)，它的内容可以通过bridge fdb命令可以查看。

#### host-gw模式

前面的两种模式都是二层网络的解决方案，对于三层网络，Flannel提供host-gw解决方案。

![host-gw模式](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gw23xvdpxoj314u0mojt9.jpg)

如上所示，如果我nginx01要访问nginx02，流程如下：

1. 转发请求包会被转发到cni0；
2. 到达本机后会匹配本机的路由，如上的路由信息，然后发现要到172.20.2.0/24的请求要经过eth0出去，并且吓一跳地址为172.16.1.130；
3. 到达Node2过后，通过路由规则到node02的cni0，再转发到nginx02；

其工作流程比较简单，主要是会在节点上生成许多路由规则。 host-gw的工作原理就是将Flannel的所有子网的下一跳设置成该子网对应的宿主机的IP地址，也就是说Host会充当这条容器通信路径的网关，当然，Flannel子网和主机的信息会保存在Etcd中，flanneld进程只需要WATCH这个数据的变化，然后实时更新路由表。

在这种模式下，就免除了额外的封包解包的性能损耗，在这种模式下，性能损耗大约在10%左右，而XVLAN隧道的机制，性能损耗大约在20%~30%。

从上面可以知道，host-gw的工作核心为IP包在封装成帧发送出去的时候会在使用路由表中写下一跳来设置目的的MAC地址，这样它就会经过二层转发到达目的宿主机。这就要求集群宿主机必须是二层连通的。

### calico

Calico是Kubernetes生态系统中另一种流行的网络选择。虽然Flannel被公认为是最简单的选择，但Calico以其性能、灵活性而闻名。Calico的功能更为全面，不仅提供主机和pod之间的网络连接，还涉及[网络安全](https://cloud.tencent.com/product/ns?from=10680)和管理。在现有的calico插件中，也提供的三种模式，分别是BGP模式，RR模式和IPIP模式。下面主要说明BGP模式和IPIP模式。

![calico](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gw24xtymjmj314q0nqwg1.jpg)

#### IPIP模式

Calico 的ipip模式和flannel的vxlan差不多，也是基于二层数据的封装和解封；也会创建一个虚拟网卡 **tunl0**。在前面提到过，Flannel host-gw 模式最主要的限制，就是要求集群宿主机之间是二层连通的。而这个限制对于 Calico 来说，也同样存在。

![ipip模式](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gw253xcpa9j311y0ny75q.jpg)

如上所示, Pod 1 访问 Pod 2大致流程如下：

1. 数据包从容器1出到达Veth Pair另一端（宿主机上，以cali前缀开头）；
2. 进入IP隧道设备（tunl0），由Linux内核IPIP驱动封装在宿主机网络的IP包中（新的IP包目的地之是原IP包的下一跳地址，即192.168.31.63），这样，就成了Node1 到Node2的数据包；
3. 数据包经过路由器三层转发到Node2；
4. Node2收到数据包后，网络协议栈会使用IPIP驱动进行解包，从中拿到原始IP包；
5. 然后根据路由规则，根据路由规则将数据包转发给cali设备，从而到达容器2。

#### BGP模式

![bgp模式](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gw26386mtij30zo0lkdh5.jpg)

如上所示，Pod 1 访问 Pod 2大致流程如下：

1. 数据包从容器1出到达Veth Pair另一端（宿主机上，以cali前缀开头）；
2. 宿主机根据路由规则，将数据包转发给下一跳（网关）；
3. 到达Node2，根据路由规则将数据包转发给cali设备，从而到达容器2

其中，这里最核心的“下一跳”路由规则，就是由 Calico 的 Felix 进程负责维护的。这些路由规则信息，则是通过 BGP Client 也就是 BIRD 组件，使用 BGP 协议传输而来的。

不难发现，Calico 项目实际上将集群里的所有节点，都当作是边界路由器来处理，它们一起组成了一个全连通的网络，互相之间通过 BGP 协议交换路由规则。这些节点，我们称为 BGP Peer。

### 总结

k8s的容器虚拟化网络方案大体分为两种： 基于隧道方案和基于路由方案

**一、隧道方案**

flannel的 vxlan模式、calico的ipip模式都是隧道方案。隧道模式分为两个过程：分配网段和封包/解包两个过程

1. 分配网络:  宿主机利用etcd（etcd中维护ip）会为当前主机上运行的容器分配一个虚拟ip，并且宿主机上运行一个代理网络进程agent，代理出入的数据包。
2. 封包/解包: 宿主上的agent进程会改变容器的发出的数据包的源ip和目的ip，目的宿主机上的agent收到数据包进行拆包然后送到目的容器。

**二、路由方案**

flannel的host-gw模式，calico的bgp模式都是路由方案。路由方案也分为两个过程：分配网段、广播路由两个阶段

1. 分配网段:  类似隧道模式，每台宿主上的agent会从etcd中为每个容器分配一个虚ip。
2. 广播路由:  agent会在宿主机上增加一套路由规则，凡是目的地址是该容器的ip的就发往容器的虚拟网卡上，同时会通过BGP广播协议将自己的虚拟ip发往集群中其他node节点，其他的node节点收到广播同样在本机创建一条路由规则：该虚拟ip的数据包发至他的宿主机ip上。

**优缺点对比:**

1. 由于隧道模式存在封包和拆包的过程而路由模式没有，所以路由模式性能高于隧道模式；
2. 隧道模式通过agent代理工作在ip层而路由模型工作在mac层下；
3. 路由模式会因为路由表膨胀性能下降；

### 参考文档

- https://aijishu.com/a/1060000000081891   // fannel网络性能测试
- https://blog.csdn.net/u010230794/article/details/103608777  // 总结的参考处
- https://cloud.tencent.com/developer/article/1602843   // kubernetes中常用网络插件之Flannel
- https://zhdya.okay686.cn/2019/12/22/K8S%E9%9B%86%E7%BE%A4%E7%BD%91%E8%B7%AFCalico%E7%BD%91%E7%BB%9C%E7%BB%84%E4%BB%B6%E5%AE%9E%E8%B7%B5(BGP%E3%80%81RR%E3%80%81IPIP)/   // Calico网络组件实践(BGP、RR、IPIP)