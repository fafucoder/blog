---
title: linux 虚拟网卡解析
date: 2021-02-03 19:11:18
tags:
- linux
categories:
- linux
---

### 虚拟网卡解析

通过 `ip link` 命令可以创建多种类型的虚拟网络设备，在 `man ip link`中可以得知有以下类型的device:

```shell
bridge - Ethernet Bridge device

bond - Bonding device can - Controller Area Network interface

dummy - Dummy network interface

hsr - High-availability Seamless Redundancy device

ifb - Intermediate Functional Block device

ipoib - IP over Infiniband device

macvlan - Virtual interface base on link layer address (MAC)

macvtap - Virtual interface based on link layer address (MAC) and TAP.

vcan - Virtual Controller Area Network interface

veth - Virtual ethernet interface

vlan - 802.1q tagged virtual LAN interface

vxlan - Virtual eXtended LAN

ip6tnl - Virtual tunnel interface IPv4|IPv6 over IPv6

ipip - Virtual tunnel interface IPv4 over IPv4

sit - Virtual tunnel interface IPv6 over IPv4

gre - Virtual tunnel interface GRE over IPv4

gretap - Virtual L2 tunnel interface GRE over IPv4

ip6gre - Virtual tunnel interface GRE over IPv6

ip6gretap - Virtual L2 tunnel interface GRE over IPv6

vti - Virtual tunnel interface

nlmon - Netlink monitoring device

ipvlan - Interface for L3 (IPv6/IPv4) based VLANs

lowpan - Interface for 6LoWPAN (IPv6) over IEEE 802.15.4 / Bluetooth

geneve - GEneric NEtwork Virtualization Encapsulation

macsec - Interface for IEEE 802.1AE MAC Security (MACsec)

vrf - Interface for L3 VRF domains
```

在kubernetes容器网络虚拟化中，常用的网卡类型包括macvlan, ipvlan, veth, ipip, vlan, macvtap, bridge等。通过 `ip link add`命令能够快速的创建一个虚拟网卡。

```
ip link add ethx type dummy
```

#### macvlan

macvlan 本身是 linxu kernel 模块，其功能是允许在同一个物理网卡上配置多个 MAC 地址，即多个 interface，每个 interface 可以配置自己的 IP。**macvlan 本质上是一种网卡虚拟化技术(最大优点是性能极好)**

可以在linux命令行执行`lsmod | grep macvlan` 查看当前内核是否加载了该driver；如果没有查看到，可以通过`modprobe macvlan`来载入，然后重新查看。内核代码路径/drivers/net/macvlan.c

Macvlan 允许你在主机的一个网络接口上配置多个虚拟的网络接口，这些网络 `interface` 有自己独立的 MAC 地址，也可以配置上 IP 地址进行通信。Macvlan 下的虚拟机或者容器网络和主机在同一个网段中，共享同一个广播域。Macvlan 和 `Bridge` 比较相似，但因为它省去了 Bridge 的存在，所以配置和调试起来比较简单，而且效率也相对高。除此之外，Macvlan 自身也完美支持 `VLAN`。

macvlan的工作方式如下：

![macvlan](https://tva1.sinaimg.cn/large/008eGmZEly1gnalist17hj311k0himyy.jpg)

![macvlan工作原理](https://tva1.sinaimg.cn/large/008eGmZEly1gnalkw01hjj30u00um4hp.jpg)

在macvlan中，实体网卡称为父接口(parent interface), 创建出来的虚拟网卡称为子接口(sub interface)，其中子接口无法与父接口通讯 (带有子接口 的 VM 或容器无法与 host 直接通讯, 这是因为在macvlan模式设计的时候为了安全而禁止了宿主机和容器直接通信)，如果vm或者容器需要与host通讯，就必须额外建立一个 `sub interface`给 host 用。

##### macvlan 模式

macvlan支持三种模式，bridge、vepa、private，在创建的时候设置**mode XXX**。

Bridge模式：属于同一个parent接口的macvlan接口之间挂到同一个bridge上，可以二层互通（macvlan接口都无法与parent 接口互通）。

![bridge模式](https://tva1.sinaimg.cn/large/008eGmZEly1gnalvbza8pj30ke0damz6.jpg)

VPEA模式：所有接口的流量都需要到外部switch才能够到达其他接口。

![vpea mode](https://tva1.sinaimg.cn/large/008eGmZEly1gnalvz30r9j30ki0e8ace.jpg)

Private模式：接口只接受发送给自己MAC地址的报文。

![private mode](https://tva1.sinaimg.cn/large/008eGmZEly1gnalwp2p00j30ke0dc0ul.jpg)

#### 创建macvlan

```shell
# 创建macvlan 网卡
ip link add link eth0 name mac1@eth0 type macvlan mode bridge
ip link add link eth0 name mac2@eth0 type macvlan mode bridge

# 创建命名空间
ip netns add ns1
ip netns add ns2

# 把虚拟网卡移入到命名空间
ip link set mac1@eth0 netns ns1
ip link set mac2@eth0 netns ns2

# 进入netns为虚拟网卡分配ip地址
ip netns exec ns1 ip addr add 192.168.0.100/24 dev mac1@eth0
ip netns exec ns2 ip addr add 192.168.0.200/24 dev mac2@eth0

# 设置虚拟网卡状态为up
ip netns exec ns1 ip link set mac1@eth0 up
ip netns exec ns2 ip link set mac2@eth0 up

# 创建macvlan网卡，不移入到命名空间，并且添加路由
ip link add link eth0 name mac3@eth0 type macvlan mode bridge
ip addr add 192.168.0.150/24 dev mac3@eth0
ip link set mac3@eth0 up

ip route add 192.168.0.100/32 dev mac3@eth0
ip route add 192.168.0.200/32 dev mac3@eth0
```

##### 验证macvlan

```shell
# 查看ip地址
[root@master ~]# ip netns exec ns2 ifconfig
mac2@eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.200  netmask 255.255.255.0  broadcast 0.0.0.0
        inet6 fe80::1860:edff:fecd:43c1  prefixlen 64  scopeid 0x20<link>
        ether 1a:60:ed:cd:43:c1  txqueuelen 1000  (Ethernet)
        RX packets 5  bytes 322 (322.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 13  bytes 1006 (1006.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# ping ns1中的网卡
[root@master ~]# ip netns exec ns2 ping -c 5 192.168.0.100
PING 192.168.0.100 (192.168.0.100) 56(84) bytes of data.
64 bytes from 192.168.0.100: icmp_seq=1 ttl=64 time=0.090 ms
64 bytes from 192.168.0.100: icmp_seq=2 ttl=64 time=0.044 ms
64 bytes from 192.168.0.100: icmp_seq=3 ttl=64 time=0.052 ms
64 bytes from 192.168.0.100: icmp_seq=4 ttl=64 time=0.080 ms
64 bytes from 192.168.0.100: icmp_seq=5 ttl=64 time=0.053 ms

--- 192.168.0.100 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4088ms
rtt min/avg/max/mdev = 0.044/0.063/0.090/0.020 ms

# 无法ping通宿主机
[root@master ~]# ip netns exec ns2 ping 192.168.0.171
PING 192.168.0.171 (192.168.0.171) 56(84) bytes of data.
^X^C
--- 192.168.0.171 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3071ms

# 能够ping通宿主机上的mac3@eth0虚拟网卡
[root@master ~]# ip netns exec ns2 ping -c 5 192.168.0.150
PING 192.168.0.150 (192.168.0.150) 56(84) bytes of data.
64 bytes from 192.168.0.150: icmp_seq=1 ttl=64 time=0.063 ms
64 bytes from 192.168.0.150: icmp_seq=2 ttl=64 time=0.071 ms
64 bytes from 192.168.0.150: icmp_seq=3 ttl=64 time=0.062 ms
64 bytes from 192.168.0.150: icmp_seq=4 ttl=64 time=0.079 ms
64 bytes from 192.168.0.150: icmp_seq=5 ttl=64 time=0.090 ms

--- 192.168.0.150 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4104ms
rtt min/avg/max/mdev = 0.062/0.073/0.090/0.010 ms
```

#### ipvlan



#### dummy

dummy网卡是内核虚拟出来的网卡，没有实际的用途，要创建一个dummy类型的网卡只需要通过如下命令:

```shell
[root@master ~]# ip link add mydummy type dummy
[root@master ~]# ifconfig dummy
dummy0: flags=130<BROADCAST,NOARP>  mtu 1500
        ether 12:cf:8a:87:81:ed  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

创建dummy网卡后可以为dummy网卡设置ip地址：

````shell
[root@master ~]# ip addr add 192.168.200.200/24 dev mydummy
[root@master ~]# ifconfig mydummy
mydummy: flags=130<BROADCAST,NOARP>  mtu 1500
        inet 192.168.200.200  netmask 255.255.255.0  broadcast 0.0.0.0
        ether ae:de:93:50:84:2f  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
````

### 参考文档

- [macvlan网络模式下容器与宿主机互通](https://rehtt.com/index.php/archives/236)
- [macvlan和ipvlan](https://www.cnblogs.com/menkeyi/p/11374023.html)
- [Linux上的物理网卡与虚拟网络设备](https://www.lijiaocn.com/%E6%8A%80%E5%B7%A7/2017/03/31/linux-net-devices.html)

