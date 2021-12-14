---
title: 云计算基础-openvswitch技术
date: 2020-01-31 12:58:09
tags:
- linux
categories:
- linux
---

### 概念
在过去，数据中心的服务器是直接连在硬件交换机上，后来VMware实现了服务器虚拟化技术，使虚拟服务器(VMs)能够连接在虚拟交换机上，借助这个虚拟交换机，可以为服务器上运行的VMs或容器提供逻辑的虚拟的以太网接口，这些逻辑接口都连接到虚拟交换机上，有三种比较流行的虚拟交换机: VMware virtual switch, Cisco Nexus 1000V,和Open vSwitch。

OpenvSwitch(OVS)是运行在虚拟化平台上的虚拟交换机，其支持OpenFlow协议，也支持gre/vxlan/IPsec等隧道技术。在OVS之前，基于Linux的虚拟化平台比如KVM或Xen上，缺少一个功能丰富的虚拟交换机，因此OVS迅速崛起并开始在Xen/KVM中流行起来，并且应用于越来越多的开源项目，比如openstack neutron中的网络解决方案。

OpenvSwitch是一个高质量的、多层虚拟交换机，使用开源Apache2.0许可协议，由Nicira Networks开发，主要实现代码为可移植的C代码。它的目的是让大规模网络自动化可以通过编程扩展,同时仍然支持标准的管理接口和协议(例如NetFlow, sFlow, SPAN, RSPAN, CLI, LACP, 802.1ag)。

ovs支持以下特性:
- 支持NetFlow, IPFIX, sFlow, SPAN/RSPAN等流量监控协议
- 精细的ACL和QoS策略
- 可以使用OpenFlow和OVSDB协议进行集中控制
- Port bonding，LACP，tunneling(vxlan/gre/Ipsec)
- 适用于Xen，KVM，VirtualBox等hypervisors
- 支持标准的802.1Q VLAN协议
- 基于VM interface的流量管理策略
- 支持组播功能
- flow-caching engine(datapath模块)

### ovn架构
OVN部署由以下几个组件组成：
1. CMS(云管系统)。这是OVN的最终用户(通过其用户和管理员)。OVN最初的目标CMS是OpenStack。
2. 安装在一个中央位置的OVN数据库，可以是物理节点，虚拟节点，甚至是一个集群。
3. 一个或多个虚拟机管理程序（hypervisors）。`hypervisors必须运行Open vSwitch`, 任何支持的OVS的hypervisor平台都是可以接受的。
4. 零个或多个网关。 网关通过在隧道和物理以太网端口之间双向转发数据包，将基于隧道的逻辑网络扩展到物理网络。这允许非虚拟机器参与逻辑网络。网关可能是物理机，虚拟机或基于ASIC同时支持vtep模式的硬件交换机。

> hypervisor和网关一起被称为传输节点或chassis

![ovn各组件](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjlrywpijtj31fi0tqtf7.jpg)

ovn架构如下所示：
![ovn架构](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjlqyxhjdxj312b0u07nj.jpg)
- Openstack/CMS plugin 是 CMS 和 OVN 的接口，将CMS 的配置转化成 OVN 的格式写到 Northbound DB 。
- Northbound DB 存储逻辑数据，与传统网络设备概念一致，比如 logical switch，logical router，ACL，logical port。
- ovn-northd 类似于一个集中式控制器，把Northbound DB 里面的数据翻译后写到 Southbound DB 。
- Southbound DB 保存的数据和 Northbound DB 语义完全不一样，主要包含 3 类数据，一是物理网络数据，比如 HV（hypervisor）的 IP 地址，HV 的 tunnel 封装格式；二是逻辑网络数据，比如报文如何在逻辑网络里面转发；三是物理网络和逻辑网络的绑定关系，比如逻辑端口关联到哪个 HV 上面。
- ovn-controller 是 OVN 里面的 agent，类似于 neutron 里面的 ovs-agent，运行在每个 HV 上面，北向，ovn-controller 会把物理网络的信息写到 Southbound DB，南向，把 Southbound DB 保存的数据转化成 Openflow flow 配到本地的 OVS table 里面，来实现报文的转发。 
- ovs-vswitchd 和 ovsdb-server 是 OVS 的两个进程。

ovn中的信息流如下：
1. OVN中的配置数据从北向南流动。CMS通过其OVN/CMS插件，通过北向数据库将逻辑网络配置传递给ovn-northd, ovn-northd将配置信息编译为较低级别的表单(流表），并通过南行数据库将其传递到所有的chassis。
2. OVN部署中每个chassis必须配置专用于OVN使用的OpenvSwitch网桥，称为集成网桥。如果需要，系统启动脚本可以在启动ovn-controller之前创建此网桥。如果ovn-controller启动时这个网桥不存在，它会自动创建。Chassis的信息保存在Southbound DB 里面，由 ovn-controller/ovn-controller-vtep 来维护。
3. 当 ovn-controller 启动的时候，它去本地的数据库 Open_vSwitch 表里面读取`external_ids:system_id`，`external_ids:ovn-remote`，`external_ids:ovn-encap-ip` 和`external_ids:ovn-encap-type`的值，然后它把这些值写到 Southbound DB 里面的表 Chassis 和表 Encap 里面。(`external_ids:system_id表示 Chassis 名字。external_ids:ovn-remote表示 Sounthbound DB 的 IP 地址。external_ids:ovn-encap-ip表示 tunnel endpoint IP 地址, external_ids:ovn-encap-type表示 tunnel 封装类型，可以是 VXLAN/Geneve/STT。`)
4. OVN 支持的 tunnel 类型有三种，分别是 Geneve，STT 和 VXLAN。HV 与 HV 之间的流量，只能用 Geneve 和 STT 两种，HV 和 VTEP 网关之间的流量除了用 Geneve 和 STT 外，还能用 VXLAN.

#### OVN-North DB
NB存放的是我们定义的逻辑交换机、逻辑路由器之类的数据，我们可以通过ovn提供的命令行（ovn-nbctl）完成添加、删除、修改、查询等操作；当然可以写代码通过OVSDB协议完成类似动作。OVN的NB是面向“上层应用”的或者叫“云管平台（Cloud Management System，CMS）”所以叫“北向接口”。

![ovn north db](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjls26uzcnj31eg0pater.jpg)

#### OVN-Sourth DB
SB进程比较特殊它同时接受两边的“写入”，首先是运行在ovn-host上的ovn-controller启动之后会去主动连接到ovn-central节点上的SB进程，把自己的IP地址（Chassis），本机的OVS状态（Datapath_Binding）写入到SB数据库中（所以叫南向接口）。ovn-controller还“监听”（etcd、zookeeper类似的功能）SB数据库中流表的变化（Flow）去更新本地的OVS数据库，这叫“流表下发”。

SB中的流表是由运行在ovn-central节点上的ovn-northd进程修改的，ovn-northd会“监听”NB的改变，把逻辑交换机、路由器的定义转换成流表（Flow）写入到SB数据库。

![ovn sourth db](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjls4214vwj31e80ogq8y.jpg)

### ovs架构
OVS整体架构中，用户空间主要组件有数据库服务ovsdb-server和守护进程ovs-vswitchd。内核空间中有datapath内核模块。最上面的Controller表示OpenFlow控制器，控制器与OVS是通过OpenFlow协议进行连接。
![ovs整体架构](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjlstyacbtj31ay0f2q5p.jpg)

#### ovsdb-server
ovsdb-server 是ovs轻量级的数据库服务，用于存储整个 OvS 的配置信息，包括接口，交换内容，VLAN，虚拟交换机的创建，网卡的添加等信息与操作记录。都被 ovsdb 保存到一个 conf.db 文件（JSON 格式）里面，通过 db.sock 提供服务。OvS 主进程 ovs-vswitchd 根据数据库中的配置信息工作。
```
ovsdb-server /etc/openvswitch/conf.db 
    -vconsole:emer -vsyslog:err 
    -vfile:info 
    --remote=punix:/var/run/openvswitch/db.sock 
    --private-key=db:Open_vSwitch,SSL,private_key 
    --certificate=db:Open_vSwitch,SSL,certificate 
    --bootstrap-ca-cert=db:Open_vSwitch,SSL,ca_cert --no-chdir 
    --log-file=/var/log/openvswitch/ovsdb-server.log 
    --pidfile=/var/run/openvswitch/ovsdb-server.pid
    --detach 
    --monitor
```
其中：
- /etc/openvswitch/conf.db：是数据库文件存放位置，ovsdb-server 需要该文件才能启动，可以使用 ovsdb-tool create 命令创建并初始化此数据库文件。
- --remote=punix:/var/run/openvswitch/db.sock：实现了一个 Unix Sockets 连接，OvS 主进程 ovs-vswitchd 或其它命令工具（e.g. ovsdb-client） 通过这个 Socket 连接来管理 ovsdb。
- /var/log/openvswitch/ovsdb-server.log：ovsdb-server 的运行日志文件。

#### ovs-vswitchd
ovs-vswitchd 本质是一个守护进程，是 OvS 的核心部件。ovs-vswitchd 和 Datapath 一起实现 OvS 基于流表（Flow-based Switching）的数据交换。它通过 OpenFlow 协议可以与 OpenFlow 控制器通信，使用 ovsdb 协议与 ovsdb-server 数据库服务通信，使用 netlink 和 Datapath 内核模块通信。ovs-vswitchd 支持多个独立的 Datapath，ovs-vswitchd 需要加载 Datapath 内核模块才能正常运行。ovs-vswitchd 在启动时读取 ovsdb-server 中的配置信息，然后自动配置 Datapaths 和 OvS Switches 的 Flow Tables，所以用户不需要额外的通过执行` ovs-dpctl `指令工具去操作 Datapath。当 ovsdb 中的配置内容被修改，ovs-vswitched 也会自动更新其配置以保持数据同步。ovs-vswitchd 也可以从 OpenFlow 控制器获取流表项。

在OVS中，ovs-vswitchd从OpenFlow控制器获取流表规则，然后把从datapath中收到的数据包在流表中进行匹配，找到匹配的flows并把所需应用的actions返回给datapath，同时作为处理的一部分，ovs-vswitchd会在datapath中设置一条datapath flows用于后续相同类型的数据包可以直接在内核中执行动作，此datapath flows相当于OpenFlow flows的缓存。对于datapath来说，其并不知道用户空间OpenFlow的存在

#### 工作原理
![ovs工作原理](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjlt5f3oxhj316k0okdpu.jpg)

1. 内核态的 Datapath 监听接口设备流入的数据包。
2. 如果 Datapath 在内核态流表缓存没有找到相应的匹配流表项则将数据包传入（upcall）到用户态的 ovs-vswitchd 守护进程处理。
3. （可选）用户态的 ovs-vswitchd 拥有完整的流表项，通过 OpenFlow 协议与 OpenFlow 控制器或者 ovs-ofctl 命令行工具进行通信，主要是接收 OpenFlow 控制器南向接口的流表项下发。或者根据流表项设置，ovs-vswitchd 可能会将网络包以 Packet-In 消息发送给 OpenFlow 控制器处理。
4. ovs-vswitchd 接收到来自 OpenFlow 控制器或 ovs-ofctl 命令行工具的消息后会对内核态的 Flow Table 进行更新。或者根据局部性原理，用户态的 ovs-vswitchd 会将刚刚执行过的 Datapath 没有缓存的流表项注入到 Flow Table 中。
5. ovs-vswitchd 匹配完流表项之后将数据包重新注入（reinject）到 Datapath。
6. Datapath 再次访问 Flow Table 获取流表项进行匹配。
7. 最后，网络包被 Datapath 根据流表项 Actions 转发或丢弃。

上述，Datapath 和 ovs-vswitchd 相互配合中包含了两种网络包的处理方式：

- Fast Path：Datapatch 加载到内核后，会在网卡上注册一个钩子函数，每当有网络包到达网卡时，这个函数就会被调用，将网络包开始层层拆包（MAC 层，IP 层，TCP 层等），然后与流表项匹配，如果找到匹配的流表项则根据既定策略来处理网络包（e.g. 修改 MAC，修改 IP，修改 TCP 端口，从哪个网卡发出去等等），再将网络包从网卡发出。这个处理过程全在内核完成，所以非常快，称之为 Fast Path。
- Slow Path：内核态并没有被分配太多内存，所以内核态能够保存的流表项很少，往往有新的流表项到来后，老的流表项就被丢弃。如果在内核态找不到流表项，则需要到用户态去查询，网络包会通过 netlink（一种内核态与用户态交互的机制）发送给 ovs-vswitchd，ovs-vswitchd 有一个监听线程，当发现有从内核态发过来的网络包，就进入自己的处理流程，然后再次将网络包重新注入到 Datapath。显然，在用户态处理是相对较慢的，故称值为 Slow Path。

![datapath](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjlt8dpposj30o00icqbu.jpg)

在用户态的 ovs-vswtichd 不需要吝啬内存，它包含了所有流表项，这些流表项可能是 OpenFlow 控制器通过 OpenFlow 协议下发的，也可能是 OvS 命令行工具 ovs-ofctl 设定的。ovs-vswtichd 会根据网络包的信息层层匹配，直到找到一款流表项进行处理。如果实在找不到，则一般会采用默认流表项，比如丢弃这个包。

当最终匹配到了一个流表项之后，则会根据 “局部性原理（局部数据在一段时间都会被频繁访问，是缓存设计的基础原理）” 再通过 netlink 协议，将这条策略下发到内核态，当这条策略下发给内核时，如果内核的内存空间不足，则会开始淘汰部分老策略。这样保证下一个相同类型的网络包能够直接从内核匹配到，以此加快执行效率。由于近因效应，接下来的网络包应该大概率能够匹配这条策略的。例如：传输一个文件，同类型的网络包会源源不断的到来。

![openflow](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjlt9rr3w9j30l00lawmv.jpg)

### 常见命令
OpenVswitch 有许多命令，分别有不同的作用，大致如下：
![OpenvSwitch命令组件](https://tva1.sinaimg.cn/large/007S8ZIlly1gjloh0399ej314q0l8tp0.jpg)

- `ovs-vsctl` 用于控制ovs db
- `ovs-ofctl` 用于管理OpenFlow switch中的流表flow
- `ovs-dpctl` 用于管理ovs的datapath
- `ovs-appctl` 用于管理ovs daemon

### 参考文档
##### 架构
- https://zhuanlan.zhihu.com/p/105582483
- https://www.cnblogs.com/liuhongru/p/11121731.html
- https://blog.csdn.net/ptmozhu/article/details/78644825  //值得一读
- https://www.cnblogs.com/gaozhengwei/p/7099928.html  //推荐阅读

##### ovs原理
- https://opengers.github.io/openstack/openstack-base-use-openvswitch/ //open vswitch详解， 非常值得阅读
- https://blog.csdn.net/Jmilk/article/details/86989975　//关于ovs的详细内容， 比上面的更加详细
- https://tonydeng.github.io/sdn-handbook/ovs/internal.html //源码级别
- https://www.cnblogs.com/popsuper1982/p/8948016.html  //源码级别

##### ovs命令
- https://blog.csdn.net/rocson001/article/details/73163041  //ovs-ofctl和ovs-vsctl
- http://www.rendoumi.com/open-vswitchde-ovs-vsctlming-ling-xiang-jie //ovs-vsctl vlan
- https://segmentfault.com/a/1190000019612525?utm_source=tag-newest //ovs docker
- https://www.sdnlab.com/sdn-guide/14747.html //ovs-vsctl 基础
