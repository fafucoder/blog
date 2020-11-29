title: https与http
date: 2020-08-03 22:27:36
tags:

- linux
categories:
- linux

## 问题定义

在Docker 作为高级容器引擎快速发展的同时，Google也开始将自身在容器技术及集群方面的积累贡献出来。 随着kubernetes的流行，越来越多的企业通过kubernetes部署应用。为了应对越来越多的企业对上云的需求，我司推出了PacificXOS平台以满足企业上云的需求。本文档针对PacificXOS的规格要求，主要针对了一下需求。

### 多租户要求

在kubernetes中，namespace 是对一组资源和对象的抽象集合，可以用来将系统内部的对象划分为不通的项目和用户组， 在namespace下可以跑一个个pod, pod作为kubernetes资源分配的最小单元。在kubernetes的网络模型中，不同命名空间下的pod能够直接互通，因此在k8s中namespace只是做了逻辑上的隔离。但是在实际的企业生产中，不同namespace下的pod期望能够实现网络隔离。因此在PacificXOS的网络插件实现中，需要实现租户隔离的功能。

### 固定IP问题

在kubernetes中，当pod创建时，kubelet通过cni接口调用网络插件，为pod分配ip地址，当pod被删除的时候，通过网络插件回收ip地址。因此，当pod被删除重建时，分配到的ip地址可能与上次不一致。但是企业生产中，期望拥有固定ip的能力，例如当部署mysql应用时，mysql能够获得固定的ip地址，其他应用可以直接通过这个ip地址访问数据库。因此，PacificXOS的网络插件需要具有固定ip的能力。

### 服务质量QoS

在服务器上，网络资源带宽总是有限的，如果采用平等的原则的话，某些较为重要的应用就无法或者更多的带宽资源。为了实现较高优先级的应用获取更多带宽资源，PacificXOS上需要提供有QoS的能力，通过对ingress跟egress的带宽限制，pod能实时的调整带宽资源，而无需重启pod。

## 设计考虑

OpenvSwitch(OVS)是运行在虚拟化平台上的虚拟交换机，其支持OpenFlow协议，也支持gre/vxlan/IPsec等隧道技术。在OVS之前，基于Linux的虚拟化平台比如KVM或Xen上，缺少一个功能丰富的虚拟交换机，因此OVS迅速崛起并开始在Xen/KVM中流行起来，并且应用于越来越多的开源项目，比如openstack neutron中的网络解决方案。ovs中支持以下特性:

(1)支持NetFlow, IPFIX, sFlow, SPAN/RSPAN等流量监控协议

(2) 精细的ACL和QoS策略

(3) 可以使用OpenFlow和OVSDB协议进行集中控制

(4) Port bonding，LACP，tunneling(vxlan/gre/Ipsec)

(5) 适用于Xen，KVM，VirtualBox等hypervisors

(6) 支持标准的802.1Q VLAN协议

(6) 基于VM interface的流量管理策略

(7) 支持组播功能

(8) flow-caching engine(datapath模块)

因此，通过平移ovs的网络能力到kubernetes中，kubernetes作为上层的CMS，通过与OVN交互，最终实现网络规则要求。

![ovn架构](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjlqyxhjdxj312b0u07nj.jpg)

选择基于kube-ovn作为PacificXOS的网络实现主要基于以下的考虑。

### 高可用考虑

PacifixXOS的网络实现中，ovn-central组件负责创建ovn-nb， ovn-sb数据库，对外提供接口,ovn-controller组件通过接口连接数据库，最终把数据下发到各个节点的ovs中。如果ovn-central组件不可用的时候，意味着整个集群网络不可用，当ovn-controller不可用时，意味着无法下发流表到ovs中，故而需要考虑的高可用组件包括ovn-central, ovn-controller。

![image-20201125143020298](/Users/dawn/Library/Application Support/typora-user-images/image-20201125143020298.png)

####  OVN-Central组件高可用

​	在ovn-central组件高可用中，首先通过kube-ovn/role=master标记出ovn-central部署的节点。紧接着ovn-central会获取所有到标记节点的node_IP，接着部署在各个节点的ovn-central组件会通过db-nb-cluster-remote-addr，db-sb-cluster-remote-addr参数进行raft选举，选出leader节点，然后启用ovn-nb，ovn-sb数据库。

​	当leader选举，数据库创建完之后，进行可用性检测，可用性没问题之后会标记出leader节点，kubernetes的service通过此标记选择backend,对外提供连接数据库的接口。通过利用kubernetes的livenessProbe定时进行存活检测，当发现数据库不可用时，重新进行raft选择新的可用节点对外提供服务，进而保证组件的高可用。

​	由于采用raft一致性算法，因此在ovn-central高可用模式下，kubernetes集群的节点数最好需要三个节点以上。

![image-20201125150349081](/Users/dawn/Library/Application Support/typora-user-images/image-20201125150349081.png)

#### OVN Controller组件高可用

​		在kubernetes中，deployment controller 会动态调整pod的数量与replicaset的一致，因此通过把ovn-controller部署为deployment, 调整replicaset数量能够提供高可用方案，多个ovn-centroller通过 leader-election选举出active的pod，对下连接ovn-db数据库，对上提供数据库的CURD操作。

### 安全考虑

##### 集中式网关

​	在kube-ovn中，默认采用分布式网关的出网方式，即pod访问外网时，流量默认经过的是当前pod所在的node节点上的ovn0网卡。使用分布式网关的设计方式让跨集群流量分析变得困难，因此我们需要设计中集中式网关，即所有的流量通过走到指定的ovn0网卡上，然后经过主机网络，当出现问题是，我们可以通过tcpdump抓取指定的流量，也方便流量审计跟白名单操作等。

##### 流量镜像

​		对容器中流量抓取是一件很繁琐的事情，好在ovs中提供了流量镜像能力， Kube-OVN 会在每台主机上创建一块 mirror0 的网卡，并设置镜像规则，将该主机上所有的容器流量复制一份到 mirror0 网卡。这样我们只需要 tcpdump -i mirror0 就可以捕获到这台主机上所有的容器网络流量，极大的方便进行流量监控。

![image-20201125155952488](/Users/dawn/Library/Application Support/typora-user-images/image-20201125155952488.png)

### 可测试性设计考虑

xxx

### 可用性设计考虑

##### 集群内应用互通

​		当ovn-controller启用之后，首先需要进行初始化操作，创建一个全局的ovn-router 虚拟机路由器，当创建不同子网的时候，创建逻辑交换机ovn-switch,并且绑定到路由器上，当创建pod的时候，会为pod创建veth-pair对，veth-pair一端连接着逻辑交换机，一端连接着pod。 所以当在同一逻辑交换机上的pod互联访问的时候，数据通过veth-pair到达switch, 再经过veth-pair到达另一个pod。当处于不同逻辑交换机上的pod互联访问的时候，数据通过switch之后，接着到达路由器，接着再转发到另外一台交换机上，通过ovn-router实现了集群内不同switch。![image-20201125161916677](/Users/dawn/Library/Application Support/typora-user-images/image-20201125161916677.png)

##### 集群与主机网络互通

​		由于集群内pod走隧道网络，因此集群内的应用无法与主机网络互通，为了实现node与pod可以直接通信， 添加一个逻辑交换机, 并且在每个node节点上创建一个虚拟网卡ovn0接入到交换机上，通过该虚拟网卡完成主机与pod之间的互通。

![image-20201125165354364](/Users/dawn/Library/Application Support/typora-user-images/image-20201125165354364.png)

​		创建逻辑交换机的时候，会在每个node节点的主机网络上添加一条路由规则， pod要访问外网时，数据首先到达ovn0网卡，然后经过iptables规则通过主机网络出网。当数据包到达主机网络时，ipset会匹配podIP是否在规则中，如果在规则中进行iptables的MASQUERADE动作，即改写封包来源 IP 为NodeIP。pod接收到外部网络的数据包跟出网的方式类似，首先通过路由表把数据转到ovn0网卡上，进而进到ovs网络中到达pod。

![iptables](https://tva1.sinaimg.cn/large/0081Kckwly1gkbtznc2uaj319r0u0aj0.jpg)

### 易用性设计考虑

对用户只提供两个接口，分别是子网跟IP接口。 每当通过子网接口创建子网的时候，对应的就创建逻辑交换机，子网中绑定了命名空间namespace, 多个namespace 可以共享同一子网。 创建pod的时候，pod通过与namespace进行关联，进而从关联的子网获取ip地址。当pod获取到ip地址时可以通过ip接口获取地址的详细信息，包括mac地址，网关等信息。

### 可修改及兼容性考虑

子网能够动态的调整绑定的命名空间namespace, pod绑定的项目发生改变时，pod重启后，能够从新的子网中获取到地址。

### 主要结构

![kube-ovn系统架构](http://dockone.io/uploads/article/20190710/decaf6ff139442ffeccd70b7ad3d0bc3.png)

#### ovn-central组件

- 功能

创建ovn，ovs数据库

#### ovs组件

- 功能

创建ovs数据库

- 

#### kube-ovn-controller组件

#### kube-ovn-cniserver组件



![](/Users/dawn/Library/Application Support/typora-user-images/image-20201126210612825.png)

#### ip地址分配

![](/Users/dawn/Library/Application Support/typora-user-images/image-20201126210657468.png)

#### 数据流

![image-20201126205703690](/Users/dawn/Library/Application Support/typora-user-images/image-20201126205703690.png)