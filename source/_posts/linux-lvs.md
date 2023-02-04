---
title: linux lvs原理及实践
date: 2021-12-19 17:01:20
tags:
 - linux
categories:
- linux
---

### 前言

负载均衡(Load Balance)的职责是将网络请求，或者其他形式的负载“均摊”到不同的机器上，让每台服务器获取到适合自己处理能力的负载。在为高负载服务器分流的同时，还可以避免资源浪费。负载均衡的原理就是当用户的请求到达前端负载均衡器(Director Server)时，通过设置好的调度算法，智能均衡的将请求分发到后端真正服务器上(Real Server)。根据请求类型的不同可以将负载均衡分为四层负载均衡(L4)和七层负载均衡(L7)， 常见的负载均衡器包括LVS，Nginx, HAProxy等。

### LVS概述

LVS是Linux Virtual Server的简称， 也就是 Linux 虚拟服务器，工作在 OSI 模型的传输层，即四层负载均衡。LVS主要由两部分组成，包括ipvs和ipvsadm。

- ipvs(ip virtual server)：工作在内核空间，是真正生效实现调度的代码。
- ipvsadm：工作在用户空间，叫 ipvsadm，负责为 ipvs 内核框架编写规则，定义谁是集群服务，而谁是后端真实的服务器(Real Server)

### LVS 基本工作原理

LVS的底层是利用NETIFILTER的钩子能力：

![ipvs工作原理](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gxv02bh3vzj30ow0c5dgv.jpg)

1. 当用户向负载均衡调度器（Director Server）发起请求，调度器将请求发往至内核空间
2. PREROUTING 链首先会接收到用户请求，判断目标 IP 确定是本机 IP，将数据包发往 INPUT 链
3. IPVS 是工作在 INPUT 链上的，当用户请求到达 INPUT 时，IPVS 会将用户请求和自己已定义好的集群服务进行比对，如果用户请求的就是定义的集群服务，那么此时 IPVS 会强行修改数据包里的目标 IP 地址及端口，并将新的数据包发往 POSTROUTING 链
4. POSTROUTING 链接收数据包后发现目标 IP 地址刚好是自己的后端服务器，那么此时通过选路，将数据包最终发送给后端的服务器

### LVS 调度算法

前言说到负载均衡器原理是根据负载均衡算法把请求转发到后端真实服务器上。LVS作为四层负载均衡器，针对不同的网络服务需求和服务器配置，LVS调度器实现了多种负载均衡调度算法，主要包括如下：

1. **轮询调度 rr（Round Robin）**：这种算法是最简单的，就是按依次循环的方式将请求调度到不同的服务器上，该算法最大的特点就是简单。轮询算法假设所有的服务器处理请求的能力都是一样的，调度器会将所有的请求平均分配给每个真实服务器，不管后端 RS 配置和处理能力，非常均衡地分发下去。

2. **加权轮叫 wrr（Weighted Round Robin）**：这种算法比 rr 的算法多了一个权重的概念，可以给 RS 设置权重，权重越高，那么分发的请求数越多，权重的取值范围 0 – 100。主要是对 rr 算法的一种优化和补充，LVS 会考虑每台服务器的性能，并给每台服务器添加要给权值，如果服务器 A 的权值为 1，服务器 B 的权值为 2，则调度到服务器 B 的请求会是服务器 A 的 2 倍。权值越高的服务器，处理的请求越多。

3. **最少链接 lc（Least Connections）**：这个算法会根据后端 RS 的连接数来决定把请求分发给谁，比如 RS1 连接数比 RS2 连接数少，那么请求就优先发给 RS1。

4. **加权最少链接 wlc（Weighted Least Connections）**：这个算法比最少链接 lc算法多了一个权重的概念。

5. **基于局部性的最少链接 lblc（Locality-Based Least Connections）**：这个算法是针对目标 IP 地址的负载均衡，目前主要用于 Cache 集群系统。该算法根据请求的目标 IP 地址找出该目标 IP 地址最近使用的服务器，若该服务器 是可用的且没有超载，将请求发送到该服务器；若服务器不存在，或者该服务器超载且有服务器处于一半的工作负载，则用 “最少链接” 的原则选出一个可用的服务 器，将请求发送到该服务器。

6. **复杂的基于局部性最少的连接算法 lblcr（Locality-Based Least Connections with Replication）***：这个算法也是针对目标 IP 地址的负载均衡，目前主要用于 Cache 集群系统。它与 LBLC 算法的不同之处是它要维护从一个 目标 IP 地址到一组服务器的映射，而 LBLC 算法维护从一个目标 IP 地址到一台服务器的映射。该算法根据请求的目标 IP 地址找出该目标 IP 地址对应的服务 器组，按 “最小连接” 原则从服务器组中选出一台服务器，若服务器没有超载，将请求发送到该服务器，若服务器超载；则按 “最小连接” 原则从这个集群中选出一 台服务器，将该服务器加入到服务器组中，将请求发送到该服务器。同时，当该服务器组有一段时间没有被修改，将最忙的服务器从服务器组中删除，以降低复制的 程度。

7. **目标地址散列 dh（Destination Hashing）**：这个算法根据请求的目标 IP 地址，作为散列键（Hash Key）从静态分配的散列表找出对应的服务器，若该服务器是可用的且未超载，将请求发送到该服务器，否则返回空。

8. **源地址散列 sh（Source Hashing）**：这个调度算法根据请求的源 IP 地址，作为散列键（Hash Key）从静态分配的散列表找出对应的服务器，若该服务器是可用的且未超载，将请求发送到该服务器，否则返回空。

### LVS 工作模式

根据负载均衡器对数据包的处理方式分类，LVS支持三种工作模式，分别为NAT模式，DR模式以及TUN模式。原生的LVS工作模式不支持FULLNAT，FULLNAT模式需要自己重新编译LVS。在介绍LVS工作模式之前有必要先解释下相关名词：

- DS：Director Server，指的是前端负载均衡器节点。
- RS：Real Server，后端真实的工作服务器。
- VIP：向外部直接面向用户请求，作为用户请求的目标的 IP 地址。
- DIP：Director Server IP，主要用于和内部主机通讯的 IP 地址。
- RIP：Real Server IP，后端服务器的 IP 地址。
- CIP：Client IP，访问客户端的 IP 地址。

#### LVS NAT模式

##### LVS NAT的工作原理

1. 当用户请求到达 Director Server，此时请求的数据报文会先到内核空间的 PREROUTING 链。 此时报文的源 IP 为 CIP，目标 IP 为 VIP
2. PREROUTING 检查发现数据包的目标 IP 是本机，将数据包送至 INPUT 链
3. IPVS 比对数据包请求的服务是否为集群服务，若是，修改数据包的目标 IP 地址为后端服务器 IP，然后将数据包发至 POSTROUTING 链。 此时报文的源 IP 为 CIP，目标 IP 为 RIP
4. POSTROUTING 链通过选路，将数据包发送给 Real Server
5. Real Server 比对发现目标为自己的 IP，开始构建响应报文发回给 Director Server。 此时报文的源 IP 为 RIP，目标 IP 为 CIP
6. Director Server 在响应客户端前，此时会将源 IP 地址修改为自己的 VIP 地址，然后响应给客户端。 此时报文的源 IP 为 VIP，目标 IP 为 CIP

![LVS NAT模式](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gxv0a244zyj30ou0dtmys.jpg)

##### LVS NAT模式特点

- **RS 应该使用私有地址，RS 的网关必须指向 DIP。**
- **DIP 和 RIP 必须在同一个网段内。**
- 支持端口映射。
- RS 可以使用任意操作系统。
- 请求和响应报文都需要经过 Director Server，高负载场景中，Director Server 易成为性能瓶颈。

##### LVS NAT模式实践

一、使用Docker模拟(适用于没虚拟机的情况)

```shell
# 1. 起两个nginx容器，分别是 
[root@localhost ~]# docker run -d -p 8000:8000 --name first -t jwilder/whoami
[root@localhost ~]# docker run -d -p 8001:8000 --name second -t jwilder/whoami

# 2. 获取容器的IP地址
[root@localhost ~]# docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' first
[root@localhost ~]# docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' second

# 执行curl获取结果(ip为步骤二返回的结果)
[root@localhost ~]# curl 172.17.0.2:8000
[root@localhost ~]# curl 172.17.0.3:8000

# 创建ipvs规则
[root@localhost ~]# ipvsadm -A -t 192.168.56.102:80 -s rr
[root@localhost ~]# ipvsadm -a -t 192.168.56.102:80 -r 172.17.0.2:8000 -m
[root@localhost ~]# ipvsadm -a -t 192.168.56.102:80 -r 172.17.0.3:8000 -m
[root@localhost ~]# ipvsadm -Ln
TCP  192.168.56.102:80 rr
  -> 172.17.0.2:8000              Masq    1      0          0         
  -> 172.17.0.3:8000              Masq    1      0          0   
# 验证是否生效(返回结果一次是A，一次是B)
[root@localhost ~]# curl 192.168.56.102
I'm fdb291a03b87
[root@localhost ~]# curl 192.168.56.102
I'm 9799eb62225c
```

二、使用虚拟机模拟

```shell
# 前置条件：
#   1. 虚拟机网段一致
#   2. 服务器2，3上部署了http服务器,且内容不一致

# 在虚拟机1上执行， (本机IP为192.168.56.101)
ipvsadm -A -t 192.168.57.4:80 -s rr
ipvsadm -a -t 192.168.57.4:80 -r 192.168.56.102:80 -m
ipvsadm -a -t 192.168.57.4:80 -r 192.168.56.104:80 -m

## 开启ip forward
echo 1 >/proc/sys/net/ipv4/ip_forward

# 在虚拟机2，3上执行，(本机IP为192.168.56.102， 192.168.56.104), 如果默认路由不设置为虚拟机1，发现是访问不通的
ip route del default
ip route add default via 192.168.56.101

# 检查是否能够访问, 在虚拟机1上执行curl
curl 192.168.57.4
```

#### LVS DR模式

DR（Direct Routing 直接路由模式）模式时 LVS 调度器只接收客户发来的请求并将请求转发给后端服务器，后端服务器处理请求后直接把内容直接响应给客户，而不用再次经过 LVS 调度器。LVS 只需要将网络帧的 MAC 地址修改为某一台后端服务器 RS 的 MAC，该包就会被转发到相应的 RS 处理，注意此时的源 IP 和目标 IP 都没变。RS 收到 LVS 转发来的包时，链路层发现 MAC 是自己的，到上面的网络层，发现 IP 也是自己的，于是这个包被合法地接受，RS 感知不到前面有 LVS 的存在。而当 RS 返回响应时，只要直接向源 IP（即用户的 IP）返回即可，不再经过 LVS。

##### LVS DR模式工作原理

1. 当用户请求到达 Director Server，此时请求的数据报文会先到内核空间的 PREROUTING 链。 此时报文的源 IP 为 CIP，目标 IP 为 VIP。
2. PREROUTING 检查发现数据包的目标 IP 是本机，将数据包送至 INPUT 链。
3. IPVS 比对数据包请求的服务是否为集群服务，若是，将请求报文中的源 MAC 地址修改为 DIP 的 MAC 地址，将目标 MAC 地址修改 RIP 的 MAC 地址，然后将数据包发至 POSTROUTING 链。 此时的源 IP 和目的 IP 均未修改，仅修改了源 MAC 地址为 DIP 的 MAC 地址，目标 MAC 地址为 RIP 的 MAC 地址。
4. 由于 DS 和 RS 在同一个网络中，所以是通过二层来传输。POSTROUTING 链检查目标 MAC 地址为 RIP 的 MAC 地址，那么此时数据包将会发至 Real Server。
5. RS 发现请求报文的 MAC 地址是自己的 MAC 地址，就接收此报文。处理完成之后，将响应报文通过 lo 接口传送给 eth0 网卡然后向外发出。 此时的源 IP 地址为 VIP，目标 IP 为 CIP。
6. 响应报文最终送达至客户端。

![LVS DR模式](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gxvmcgh1xwj30oy0dwq57.jpg)

##### LVS DR模式特点

- 保证前端路由将目标地址为 VIP 报文统统发给 Director Server，而不是 RS。
- RS 可以使用私有地址；也可以是公网地址，如果使用公网地址，此时可以通过互联网对 RIP 进行直接访问。
- VIP与DIP，RIP可以不在同一网段中.
- 所有的请求报文经由 Director Server，但响应报文必须不能进过 Director Server。
- 不支持地址转换，也不支持端口映射。
- RS 可以是大多数常见的操作系统。
- RS 的网关绝不允许指向 DIP（因为我们不允许他经过 Director Server）。
- RS 上的 lo 接口配置 VIP 的 IP 地址。
- **RS 和 DS 必须在同一个物理网络中(同一机房中)。**

##### LVS DR模式的相关问题

为了解决`保证前端路由将目标地址为 VIP 报文统统发给 Director Server，而不是 RS。`的问题一般有如下解决方案：

- 在前端路由器做静态地址路由绑定，将对于 VIP 的地址仅路由到 Director Server。但用户未必有路由操作权限，因为有可能是运营商提供的，所以这个方法未必实用。
- arptables：在 arp 的层次上实现在 ARP 解析时做防火墙规则，过滤 RS 响应 ARP 请求。这是由 iptables 提供的。
- 修改 RS 上内核参数（`arp_ignore` 和 `arp_announce`）将 RS 上的 VIP 配置在 lo 接口的别名上，并限制其不能响应对 VIP 地址解析请求。

##### 关于arp_ignore和arp_announce

arp_ignore：定义接收 ARP 请求时的响应级别 

- 0：响应任意网卡上接收到的对本机 IP 地址的 ARP 请求（包括环回网卡），不论目的 IP 地址是否在接收网卡上
- 1：只响应目的 IP 地址为接收网卡地址的 ARP 请求 
- 2：只响应目的 IP 地址为接收网卡地址的 ARP 请求，且 ARP 请求的源 IP 地址必须和接收网卡的地址在同网段

```
arp_ignore - INTEGER
    Define different modes for sending replies in response to
    received ARP requests that resolve local target IP addresses:
        0 - (default): reply for any local target IP address, configured on any interface
        1 - reply only if the target IP address is local address configured on the incoming interface
        2 - reply only if the target IP address is local address configured on the incoming interface
            and both with the sender's IP address are part from same subnet on this interface
        3 - do not reply for local addresses configured with scope host,
            only resolutions for global and link addresses are replied
        4-7 - reserved
        8 - do not reply for all local addresses

    The max value from conf/{all,interface}/arp_ignore is used
    when ARP request is received on the {interface}
```

arp_announce：定义将自己地址向外通告时的通告级别 

- 0：允许任意网卡上的任意地址向外通告 
- 1：尽量仅向目标网络通告与其网络匹配的地址 
- 2：仅向与本地接口上地址匹配的网络进行通告

```
arp_announce - INTEGER
    Define different restriction levels for announcing the local source IP address
    from IP packets in ARP requests sent on interface:
        0 - (default) Use any local address, configured on any interface
        1 - Try to avoid local addresses that are not in the target's subnet for this interface.
            This mode is useful when target hosts reachable via this interface require the source IP
            address in ARP requests to be part of their logical network configured on the receiving
            interface. When we generate the request we will check all our subnets that include the
            target IP and will preserve the source address if it is from such subnet.
            If there is no such subnet we select source address according to the rules for level 2.
        2 - Always use the best local address for this target. In this mode we ignore the source
            address in the IP packet and try to select local address that we prefer for talks
            with the target host. Such local address is selected by looking for primary IP addresses
            on all our subnets on the outgoing interface that include the target IP address.
            If no suitable local address is found we select the first local address we have on the
            outgoing interface or on all other interfaces, with the hope we will receive reply
            for our request and even sometimes no matter the source IP address we announce.

    The max value from conf/{all,interface}/arp_announce is used.

    Increasing the restriction level gives more chance for receiving answer from the resolved target
    while decreasing the level announces more valid sender's information.
```

##### LVS DR模式实践

```shell
# 前置条件：
#   1. 虚拟机网段一致
#   2. 服务器2，3上部署了http服务器,且内容不一致

# 在虚拟机1上执行， (本机IP为192.168.56.101)
ip addr add 192.168.56.200/32 dev enp0s8
ipvsadm -A -t 192.168.56.200:80 -s wrr
ipvsadm -a -t 192.168.56.200:80 -r 192.168.56.102:80 -g -w 1
ipvsadm -a -t 192.168.56.200:80 -r 192.168.56.104:80 -g -w 3

## 开启ip forward
echo 1 >/proc/sys/net/ipv4/ip_forward

# 在虚拟机2，3上执行，(本机IP为192.168.56.102， 192.168.56.104)
ip addr add 192.168.56.200/32 dev lo
ip route add 192.168.56.200 dev lo
echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce

# 检查是否能够访问, 在外部主机上执行curl
curl 192.168.56.200
```

#### LVS TUN 模式

> TUN工作在三层，TAP工作在二层

##### LVS TUN模式工作原理

1. 当用户请求到达 Director Server，此时请求的数据报文会先到内核空间的 PREROUTING 链。 此时报文的源 IP 为 CIP，目标 IP 为 VIP。
2. PREROUTING 检查发现数据包的目标 IP 是本机，将数据包送至 INPUT 链。
3. IPVS 比对数据包请求的服务是否为集群服务，若是，在请求报文的首部再次封装一层 IP 报文，封装源 IP 为 DIP，目标 IP 为 RIP。然后发至 POSTROUTING 链。 此时源 IP 为 DIP，目标 IP 为 RIP。
4. POSTROUTING 链根据最新封装的 IP 报文，将数据包发至 RS（因为在外层封装多了一层 IP 首部，所以可以理解为此时通过隧道传输）。 此时源 IP 为 DIP，目标 IP 为 RIP。
5. RS 接收到报文后发现是自己的 IP 地址，就将报文接收下来，拆除掉最外层的 IP 后，会发现里面还有一层 IP 首部，而且目标是自己的 lo 接口 VIP，那么此时 RS 开始处理此请求，处理完成之后，通过 lo 接口送给 eth0 网卡，然后向外传递。 此时的源 IP 地址为 VIP，目标 IP 为 CIP。
6. 响应报文最终送达至客户端。

![LVS TUN模式](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gxvobeirpcj30p00e340q.jpg)

##### LVS TUN模式特点

- 不改变请求数据包，而是在请求数据包上新增一层 IP 首部信息。因此负载均衡器不能对端口进行转发，但可以和真实服务器不在同一局域网内，且真实服务器需要支持能够解析两层 IP 首部信息，即需要支持“IP Tunneling”或“IP Encapsulation”协议
- 真实服务器中的 VIP，只能被自己 “看见”，其他设备不可知。因此 VIP 必须绑定在真实服务器的 lo 网卡上，并且不允许将此网卡信息经过 ARP 协议对外通告
- 请求的数据包经过负载均衡器后，直接由真实服务器返回给客户端，响应数据包不需要再经过负载均衡器
- RS 的网关不会也不可能指向 DIP。

##### LVS TUN模式实践

```shell
# 前置条件：
#   2. 服务器2，3上部署了http服务器,且内容不一致

# 在虚拟机1上执行， (本机IP为192.168.56.101)
ip addr add 192.168.56.210/32 dev enp0s8
ipvsadm -A -t 192.168.56.210:80 -s rr
ipvsadm -a -t 192.168.56.210:80 -r 192.168.56.102:80 -i
ipvsadm -a -t 192.168.56.210:80 -r 192.168.56.104:80 -i

# 在虚拟机2，3上执行，(本机IP为192.168.56.102， 192.168.56.104)
modprobe ipip // 启用ipip
lsmod | grep ipip
ip addr add 192.168.56.210/32 dev tunl0
ip link set tunl0 up
ip route add 192.168.56.210 dev tunl0

echo 1 >/proc/sys/net/ipv4/conf/lo/arp_ignore
echo 2 >/proc/sys/net/ipv4/conf/lo/arp_announce
echo 1 >/proc/sys/net/ipv4/conf/all/arp_ignore
echo 2 >/proc/sys/net/ipv4/conf/all/arp_announce
echo 0 > /proc/sys/net/ipv4/conf/tunl0/rp_filter
echo 0 > /proc/sys/net/ipv4/conf/all/rp_filter

# 检查是否能够访问, 在外部主机上执行curl
curl 192.168.56.210
```

#### LVS FULLNAT模式

LVS NAT, DR模式中，RS跟DS必须在同一个VLAN中，当集群规模较小时，使用 NAT、DR 模式都是没有问题的，当集群内有几十台以上时，那么这些服务器通常都不在同一个 VLAN/网段 内了。这时，必须再研发出一种能支持跨 VLAN/网段 通信的模式，FULLNAT 模式就是为了解决这个问题而生的。

LVS FullNAT 模式几乎和 LVS NAT 模式相同，不同之处即是：引入 Local Address（内网 IP 地址）。CIP->VIP 转换换为 LIP->RIP，而 LIP 和 RIP 均为 IDC 内网 IP，因此可以跨 VLAN 通讯。LVS原生模式不支持FULLNAT，因此需要自己手动编译内核～

##### LVS FULLNAT内核编译

> 注意：内核编译的内核包需要跟自己的操作系统内核版本对应上(大版本对应上~)，暂时未找到centos7对应的内核包在哪儿。
>
> 本机的内核版本为：2.6.32-220

1. 安装依赖

```shell
yum -y install elfutils-devel.x86_64 audit-libs-devel.x86_64
yum -y install rpmdevtools yum-utils
yum -y install redhat-rpm-config
yum -y install gcc xmlto patchutils asciidoc zlib-devel binutils-devel newt-devel python-devel hmaccalc perl-ExtUtils-Embed.x86_64
yum -y install rng-tools 随机数生成器
yum -y install bison
```

2. 下载内核包，ipvs补丁包

```shell
# 内核包
wget ftp://ftp.redhat.com/pub/redhat/linux/enterprise/6Server/en/os/SRPMS/kernel-2.6.32-220.23.1.el6.src.rpm
# lvs fullnat包
wget http://kb.linuxvirtualserver.org/images/a/a5/Lvs-fullnat-synproxy.tar.gz

```

3. 生成内核代码

```shell
rpm -ivh kernel-2.6.32-220.23.1.el6.src.rpm

# 生成随机数(一定要生成随机数，否则rpmbuild的时候会卡住)
rngd -r /dev/urandom   

# rpmbuild -bp kernel.spec
cd ~/rpmbuild/SPECS

rpmbuild -bp --target=$(uname -m) kernel.spec
```

	4. 打 LVS补丁

```shell
tar -zxvf Lvs-fullnat-synproxy.tar.gz

cp ~/lvs-fullnat-synproxy/lvs-2.6.32-220.23.1.el6.patch ~/rpmbuild/BUILD/kernel-2.6.32-220.23.1.el6/linux-2.6.32-220.23.1.el6.x86_64 && cd $_

patch -p1 < lvs-2.6.32-220.23.1.el6.patch
```

5. 编译内核

```shell
# 最好不要重新修改内核版本(遇到过改了内核版本后，编译没问题，重启后触发Kernel panic – not syncing: Attempted to kill init的问题)
# 使用-j 加快内核编译速度
make -j16
make modules_install
make install
```

6. 重启操作系统

```shell
reboot

# 重启后发现系统版本已经改变了， Linux localhost.localdomain 2.6.32 #2 SMP Thu Dec 30 03:38:53 EST 2021 x86_64 x86_64 x86_64 GNU/Linux
```

7. 编译ipvsadm, keepalived等工具

```shell
cd ~/lvs-fullnat-synproxy && tar -zxvf lvs-tools.tar.gz

# 编译 keepalived
cd ~/lvs-fullnat-synproxy/tools/keepalived
./configure --with-kernel-dir="/lib/modules/`uname -r`/build" && make && make install
 
# 编译 ipvsadm
cd ~/lvs-fullnat-synproxy/tools/ipvsadm
make && make install

# 编译 quaage
cd ~/lvs-fullnat-synproxy/tools/quagga
./configure --disable-ripd --disable-ripngd --disable-bgpd --disable-watchquagga --disable-doc  --enable-user=root --enable-vty-group=root --enable-group=root --enable-zebra --localstatedir=/var/run/quagga && make && make install
```

8. 验证是否编译成功

```shell
# 输入 ipvsadm -h 可以看到有fullnat 模式

  --gatewaying   -g                   gatewaying (direct routing) (default)
  --ipip         -i                   ipip encapsulation (tunneling)
  --fullnat      -b                   fullnat mode
  --masquerading -m                   masquerading (NAT)
```

##### LVS FULLNAT模式工作原理

1. 当用户请求到达 Director Server，此时请求的数据报文会先到内核空间的 PREROUTING 链。 此时报文的源 IP 为 CIP，目标 IP 为 VIP
2. PREROUTING 检查发现数据包的目标 IP 是本机，将数据包送至 INPUT 链
3. IPVS 比对数据包请求的服务是否为集群服务，若是，修改数据包的源IP地址分发服务器IP，目标 IP 地址为后端服务器 IP，然后将数据包发至 POSTROUTING 链。 此时报文的源 IP 为 DIP，目标 IP 为 RIP
4. POSTROUTING 链通过选路，将数据包发送给 Real Server
5. Real Server 比对发现目标为自己的 IP，开始构建响应报文发回给 Director Server。 此时报文的源 IP 为 RIP，目标 IP 为 DIP
6. Director Server 在响应客户端前，此时会将源 IP 地址修改为自己的 VIP 地址，然后响应给客户端。 此时报文的源 IP 为 VIP，目标 IP 为 CIP

![LVS FULLNAT模式](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gxw0gf5xdlj30lr0ck0tc.jpg)

##### LVS FULLNAT模式实践

```shell
# 虚拟机4的IP地址为192.168.58.100， 虚拟机2，3的IP地址为192.168.56.102， 192.168.56.104, 在虚拟机上执行：
ip addr add 192.168.58.200/32 dev eth1

ipvsadm -A -t 192.168.58.200:80 -s wrr
ipvsadm -a -t 192.168.58.200:80 -r 192.168.56.102:80 -b -w 2
ipvsadm -a -t 192.168.58.200:80 -r 192.168.56.104:80 -b -w 1
ipvsadm -P -t 192.168.58.200:80 -z 192.168.58.100

# 集群外执行curl, 验证是否成功
curl 192.168.56.200
```

### 参考文档

- [LVS 负载均衡原理及安装配置详解](https://github.com/xgfone/snippet/blob/master/snippet/docs/architecture/ha-lb/lvs-lb-and-install.md)  // 推荐阅读
- [LVS 原理介绍和配置实践](https://wsgzao.github.io/post/lvs/)  // 推荐阅读
- [深入浅出 LVS 负载均衡系列（一）：NAT、FULLNAT 模型原理](https://segmentfault.com/a/1190000039819984)  //这系列文章不错
- [深入浅出 LVS 负载均衡系列（二）：DR、TUN 模型原理](https://segmentfault.com/a/1190000039932089)  //这系列文章不错
- [深入浅出 LVS 负载均衡（三）实操 NAT、DR 模型](https://segmentfault.com/a/1190000040190992)    //这系列文章不错
- [LVS FULLNAT 实战](https://www.haxi.cc/archives/LVS-FULLNAT实战.html)   // lvs fullnat 编译，样例
- [IPVS FULLNAT and SYNPROXY](http://kb.linuxvirtualserver.org/wiki/IPVS_FULLNAT_and_SYNPROXY)  // 编译 lvs fullnat
- [LVS/fullnat模式（内核编译）](https://blog.csdn.net/chao199512/article/details/81390485)
