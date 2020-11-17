---
title: linux网络相关命令十全大补丸
date: 2020-06-06 19:19:10
tags:
- linux
categories:
- linux
---

### 网卡相关
网卡相关的命令包括查看网卡状态，更改网卡ip等，主要命令包括ip， ifconfig等。
#### ifconfig 和ip

ethtool等，ifconfig属于net-tool包里面的一个命令 ， ip 是iproute2包 包含的命令，ip命令更为强大 ifconfig命令显示的信息比较好看，个人更喜欢使用ifconfig查看网卡信息等， iproute2跟net-tool的对照表如下所示:
![网络工具](linux-netowrk-command/net-tools.png)

ip 跟 ifconfig命令对照表如下:

| 操作               | ip                                                | ifconfig                                       | 备注                                                                                                                                                                        |
| -------------------- | ------------------------------------------------- | ---------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 显示所有的网卡 | ip a/ip link show                                 | ifconfig -a                                    |                                                                                                                                                                               |
| 启用或停用网卡 | ip link set eth1 down/up                          | ifconfig eth1 down/up                          |                                                                                                                                                                               |
| 显示网卡详细信息 | ip link show eth1                                 | ifconfig eth1                                  |                                                                                                                                                                               |
| 设置网卡混杂模式 | ip link set eth1 promisc on/off                   | ifconfig eth1 promisc/-promisc                 |                                                                                                                                                                               |
| 设置网卡MTU      | ip link set eth1 mtu 1400                         | ifconfig eth1 mtu 1400                         |                                                                                                                                                                               |
| 设置网卡所属命名空间 | ip link set eth1 netns pid/name                   |                                                | 关于网络命名空间，ip提供了强大的netns命令， ip netns 可查看命令详情                                                                                  |
| 配置网卡ipv4地址 | ip addr add 10.0.0.1/24 dev eth1                  | ifconfig eth1 10.0.0.1/24                      | ip添加命令dev一定要加， ip add可以添加多个地址(ip addr add 10.0.0.1/24 broadcast 10.0.0.255 dev eth1, ) ip addr add 10.0.0.2/24 broadcast 10.0.0.255 dev eth1，ifconfig则不行 |
| 修改网卡ipv4地址 | ip addr change 10.0.0.2/24 dev eth1               | ifconfig eth1 10.0.0.2/24                      | ifconfig 添加跟修改命令一致                                                                                                                                          |
| 移除网卡ipv4地址 | ip addr del 10.0.0.1/24 dev eth1                  | ifconfig eth1 0                                | ip 命令可以添加多个地址，所以删除需要对应的地址                                                                                                         |
| 配置网卡ipv6地址 | ip -6 addr add 2002:0db5:0:f102::1/64 dev eth1    | ifconfig eth1 inet6 add 2002:0db5:0:f102::1/64 |                                                                                                                                                                               |
| 修改网卡ipv4地址 | ip -6 addr change 2002:0db5:0:f102::2/64 dev eth1 | ifconfig eth1 inet6 add 2002:0db5:0:f102::2/64 |                                                                                                                                                                               |
| 移除网卡ipv6地址 | ip -6 addr del 2002:0db5:0:f102::1/64 dev eth1    | ifconfig eth1 inet6 del 2002:0db5:0:f102::1/64 |

对比以上表格命令可以得出，要修改网卡地址使用的是ip addr, 要设置网卡的配置信息使用的是ip link

网卡的命令空间操作使用 ip netns命令

#### ethtool
ethtool命令主要是查看网卡详细信息,命令比较简单`ethtool -params eth`例如
```
root@node-2:/home/dawn# ethtool enp0s9
Settings for enp0s9:
        Supported ports: [ TP ]
        Supported link modes:   10baseT/Half 10baseT/Full 
                                100baseT/Half 100baseT/Full 
                                1000baseT/Full                      //支持模式
        Supported pause frame use: No
        Supports auto-negotiation: Yes                              //支持自动协商
        Supported FEC modes: Not reported
        Advertised link modes:  10baseT/Half 10baseT/Full 
                                100baseT/Half 100baseT/Full 
                                1000baseT/Full 
        Advertised pause frame use: No
        Advertised auto-negotiation: Yes
        Advertised FEC modes: Not reported
        Speed: 1000Mb/s                                             //速率
        Duplex: Full                                                //工作模式(全双工，半双工...)
        Port: Twisted Pair
        PHYAD: 0
        Transceiver: internal
        Auto-negotiation: on                                        //自动协商打开
        MDI-X: off (auto)
        Supports Wake-on: umbg
        Wake-on: d
        Current message level: 0x00000007 (7)
                               drv probe link
        Link detected: yes
```

ethtool常见命令如下表：

| 参数 | 说明                                                                                      | 示例                        |
| ---- | ------------------------------------------------------------------------------------------- | ----------------------------- |
| -a   | 查看网卡中接收模块RX、发送模块TX和Autonegotiate模块的状态：启动on 或 停用off。 | ethtool -a enp0s9             |
| -A   | 修改网卡中 接收模块RX、发送模块TX和Autonegotiate模块的状态：启动on 或 停用off。 | ethtool -A enp0s9 rx off      |
| -i   | 显示网卡驱动的信息，如驱动的名称、版本等。                             | ethtool -i enp0s9             |
| -s   | 修改网卡的部分配置，包括网卡速度、单工/全双工模式、mac地址等    | ethtool -s enp0s9 advertise N |
| -S   | 显示NIC- and driver-specific 的统计参数，如网卡接收/发送的字节数、接收/发送的广播包个数等。 | ethtool -S enp0s9             |

### 路由相关

路由相关的命令包括防火墙，路由查看等， iptables之前已有记录，这里不做记录。

#### route
route属于net-tool包里的命令，ip route 是iproute2里面的命令。

路由格式如下：
```
root@node-2:/home/dawn# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         192.168.3.1     0.0.0.0         UG    100    0        0 enp0s3
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.3.0     0.0.0.0         255.255.255.0   U     0      0        0 enp0s3
192.168.3.1     0.0.0.0         255.255.255.255 UH    100    0        0 enp0s3
192.168.56.0    0.0.0.0         255.255.255.0   U     0      0        0 enp0s8
192.168.57.0    0.0.0.0         255.255.255.0   U     0      0        0 enp0s9
```

各字段表示如下：

| 字段      | 说明                                                     |
| ----------- | ---------------------------------------------------------- |
| Destination | 目标网段或者主机                                   |
| Gateway     | 网关地址，”*” 表示目标是本主机所属的网络，不需要路由 |
| Genmask     | 网络掩码                                               |
| Flags       | 标记。一些可能的标记如下：                    |
|             | U — 路由是活动的                                   |
|             | H — 目标是一个主机                                |
|             | G — 路由指向网关                                   |
|             | R — 恢复动态路由产生的表项                    |
|             | D — 由路由的后台程序动态地安装              |
|             | M — 由路由的后台程序修改                       |
|             | ! — 拒绝路由                                         |
| Metric      | 路由距离，到达指定网络所需的中转数（linux 内核中没有使用） |
| Ref         | 路由项引用次数（linux 内核中没有使用）     |
| Use         | 此路由项被路由软件查找的次数                 |
| Iface       | 该路由表项对应的输出接口                       |

route 命令操作: `route  [add|del] [-net|-host] target [netmask Nm] [gw Gw] [[dev] If]`

route常见的操作包括添加路由表，删除路由表等， 操作如下表:

| 操作                   | net-tool                                                                                    | iproute2 | 备注        |
| ------------------------ | ------------------------------------------------------------------------------------------- | -------- | ------------- |
| 显示所有路由       | route                                                                                       | ip route |               |
| 添加网络路由       | route add -net 192.168.55.0 netmask 255.255.255.0 eth3 / route add -net 192.168.1.0/24 eth1 |          |               |
| 添加默认路由       | route add default gw 192.168.1.1                                                            |          | 只能是gw 地址 |
| 添加主机路由       | route add -host 192.168.1.2 dev eth0                                                        |          |               |
| 添加网络路由指定默认网关 | route add -net 10.20.30.48 netmask 255.255.255.248 gw 10.20.30.41                           |          |               |
| 添加默认路由指定默认网关 | route add -host 10.20.30.148 gw 10.20.30.40                                                 |          |               |
| 删除网络路由       | route del -net 10.20.30.40 netmask 255.255.255.248 eth0                                     |          |               |
| 删除默认路由       | route del default gw 192.168.1.1                                                            |          |               |
| 删除主机路由       | route del -host 192.168.1.2 dev eth0                                                        |          |               |
| 删除网络路由指定默认网关 | route del -net 10.20.30.48 netmask 255.255.255.248 gw 10.20.30.41                           |          |               |
| 删除主机路由指定默认网关 | route del -host 10.20.30.148 gw 10.20.30.40 

### 内核模块相关
内核模块相关的命令包括查看相关模块是否加载，加载相关模块等

#### lsmod

lsmod 列出内核已载入模块的状态,例如

```
root@node-2:/home/dawn# lsmod | grep vfio
vfio_pci               45056  0
vfio_virqfd            16384  1 vfio_pci
irqbypass              16384  1 vfio_pci
vfio_iommu_type1       24576  0
vfio                   28672  2 vfio_iommu_type1,vfio_pci
```

linux 内核模块加载比较多， 一般配置grep 来检测指定模块是否加载

#### modprobe

modprobe一般加载、去除linux内核模块， 例如`modprobe vfio`, 主要命令有

- modprobe vfio: 添加vfio模块
- modprobe -r vfio: 移除vfio模块

#### modinfo

modinfo 用于查看内核模块信息, 例如

```
root@node-2:/home/dawn# modinfo vfio
filename:       /lib/modules/4.15.0-101-generic/kernel/drivers/vfio/vfio.ko
softdep:        post: vfio_iommu_type1 vfio_iommu_spapr_tce
alias:          devname:vfio/vfio
alias:          char-major-10-196
description:    VFIO - User Level meta-driver
author:         Alex Williamson <alex.williamson@redhat.com>
license:        GPL v2
version:        0.3
srcversion:     53C6B01E26FFBE3618E4286
depends:        
retpoline:      Y
intree:         Y
name:           vfio
vermagic:       4.15.0-101-generic SMP mod_unload 
signat:         PKCS#7
signer:         
sig_key:        
sig_hashalgo:   md4
parm:           enable_unsafe_noiommu_mode:Enable UNSAFE, no-IOMMU mode.  This mode provides no device isolation, no DMA translation, no host kernel protection, cannot be used for device assignment to virtual machines, requires RAWIO permissions, and will taint the kernel.  If you do not know what this is for, step away. (default: false) (bool)
```

#### lspci

lspci 用于在系统中显示有关pci总线的信息以及连接到它们的设备。

默认情况下，它显示了一个简单的设备列表。使用下面描述的选项可以请求更详细的输出或其他程序用于解析的输出。

如果要报告PCI设备驱动程序或lspci本身中的bug，请使用选项“lspci-vvx”或更好的“lspci-vvxxx”的输出(不过，可能会有警告)。

例如查看物理网卡信息：
```
root@node-2:/home/dawn# lspci | grep Eth
00:03.0 Ethernet controller: Intel Corporation 82540EM Gigabit Ethernet Controller (rev 02)
00:08.0 Ethernet controller: Intel Corporation 82540EM Gigabit Ethernet Controller (rev 02)
00:09.0 Ethernet controller: Intel Corporation 82540EM Gigabit Ethernet Controller (rev 02)

root@node-2:/home/dawn# lspci -n -s 00:09.0
00:09.0 0200: 8086:100e (rev 02)
```

lspci 常见参数如下表:

| 参数  | 描述                                                                    | 如何使用         | 备注                                      |
| ------- | ------------------------------------------------------------------------- | -------------------- | ------------------------------------------- |
| -n      | 显示pci设备的厂商和设备代码                                   | lspci -n -s 00:09.0  | 一般配置-s参数使用显示pci设备的设备驱动代码 |
| -nn     | 更加详细显示pci设备的厂商和设备代码                       | lspci -nn -s 00:09.0 |                                             |
|  -m     | 以向后兼容并且机器可读的方式转储设备信息              | lspci -m             |                                             |
| -mm     | 以机器可读的方式转储设备信息，以便脚本解析           | lspci -mm            |                                             |
|     -t  | 以树形结构显示pci设备的层次关系，包含所有总线、桥梁、设备和它们之间的连接 | lspci -t             |                                             |
| -v      | 显示所有设备的详细信息                                         | lscpi -v             |                                             |
|     -vv | 以更加详细的方式显示设备信息                                | lspci -vv            |


### 连通测试
连通测试主要测试目的主机(服务器)是否可达，常见的有telnet，traceroute，tcpdump等

#### telnet

telnet 可用于查看远方服务器端口是否开放，目的主机是否可达等,例如：

telnet 192.168.56.100 22

#### traceroute

traceroute 用于跟踪到达目的主机过程中所经过的路径,例如:

traceroute 192.168.56.100

#### tcpdump

tcpdump 用于监听网卡收包情况, tcpdump 常见命令如下表：

| 参数 | 描述                                                      | 说明                                          |
| ---- | ----------------------------------------------------------- | ----------------------------------------------- |
| -i   | 指定监听网络接口                                    | tcpdump -i eth0 (抓取eth0包)                 |
| -p   | 不将网络接口设置成混杂模式                     |                                                 |
| -c   | 指定监听数据包数量，当收到指定的包的数目后，tcpdump就会停止 |                                                 |
| -n   | 不把网络地址转换成名字                           | tcpdump -ni em2 -vv -e  | grep vlan(抓取vlan包) |
| -e   | 在输出行打印出数据链路层的头部信息         | tcpdump -ni em2 -vv -e  | grep vlan(抓取vlan包) |
| -d   | 将匹配信息包的代码以人们能够理解的汇编格式给出 |                                                 |
| -vv  | 输出详细的报文信息                                 | tcpdump -ni em2 -vv -e  | grep vlan(抓取vlan包) |

#### tcpdump expression
tcpdump expression 用于tcpdump 数据包过滤，如果不过滤的话数据包很多，不好排查，因此需要用expression 进行数据包过滤.

tcpdump利用正则表达式作为过滤报文的条件，如果数据包满足表达式的条件，则会被捕获。如果没有给出任何条件，则网络上所有的数据包将会被截获。表达式中常用关键字如下：

(1) 指定参数类型的关键字，主要包括host，net，port等，如果没有指定类型，缺省的类型是host；例如：#tcpdump host 222.24.20.86 截获ip为222.24.20.86的主机收发的所有数据包

(2) 指定数据报文传输方向的关键字，主要包括src , dst ,dst or src, dst and src，缺省为src or dst；例如：#tcpdump src net 222.24.20.1 截取源网络地址为 222.24.20.1 的所有数据包

(3) 指定协议的关键字，主要包括fddi，ip，arp，rarp，tcp，udp等类型，默认监听所有协议的数据包；例如：#tucpdump arp 截获所有arp协议的数据包

(4) 其他重要的关键字还有，gateway，broadcast，less，greater，三种逻辑运算（取非运算是 'not ','! '； 与运算是 'and ','&& '；或运算 是'or ','|| '）等。这些关键字的巧妙组合，能灵活构造过滤条件，从而满足用户需要。例如：#tcpdump host ubuntu and src port \(80 or 8080\) 截取主机ubuntu上源端口为80或8080的所有数据包。

expression 的整体结构 `expr relop expr` relop表示关系操作符，可以为>, < ,>=,<=, =, !=之一。

### 参考文档
- https://blog.csdn.net/u011956172/article/details/54617516  //net-tool跟iproute2对照表
- https://www.cnblogs.com/baiduboy/p/7278715.html //route指令详解
- https://www.cnblogs.com/jacklikedogs/p/4659249.html //modprobe, lsmod命令使用
- https://yq.aliyun.com/articles/653209 lspci命令使用
