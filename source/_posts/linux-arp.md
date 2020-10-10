---
title: arp地址解析协议
date: 2020-10-09 19:45:22
tags:
- linux
categories:
- linux
---

### 概念
ARP(Address Resolution Protocol) 即地址解析协议，用于实现从IP地址到MAC地址映射。

### ARP工作流程

#### 相同网段
![同网段ARP](https://img2018.cnblogs.com/blog/835745/201908/835745-20190812161206253-1584652320.png)

1.  PC1 要和PC3 通行，首先查看自己的ARP表，查看其中是否包含PC3的MAC地址信息，如果找到对应关系，直接利用ARP表中的MAC地址对IP数据包进行封装。并将数据包发送给PC3。

2. 如果PC1在ARP表中未找到PC3对应的MAC地址，则先缓存数据报文，然后利用广播方式（目标MAC地址FF:FF:FF:FF:FF:FF)发送一个ARP报文请求，ARP请求中的发送端MAC地址分别是PC1的IP地址和MAC地址，接收端的IP地址为PC3的IP地址，MAC地址全为0，因为ARP请求报文是以广播方式发送，所以该网段上的所有主机都可以接收到该请求包，但只有其IP地址与目的IP地址一致的PC3才会对该请求进行处理。

3. PC3将ARP请求报文中的发送端（即PC1）的IP地址和MAC地址存入自己的ARP表中。然后以单播方式向PC1发送一个ARP相应报文，应答报文中就包含了自己的MAC地址，也就是原来在请求报文中要请求的目的MAC地址。

4. PC1收到来自PC3的ARP响应报文之后，将PC3的MAC地址加入到自己的ARP表中以用于后续报文的转发，同时将原来缓存的IP数据包再次修改（在目的MAC地址字段填上PC3的MAC地址）后发送出去。

#### 跨网段
![跨网段ARP](https://img-blog.csdn.net/20160923110942379?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

1. 如果主机A不知道网关的MAC地址（也就是主机A的ARP表中没有网关对应的MAC地址表项），则主机A先在本网段中发出一个ARP请求广播，ARP请求报文中的目的IP地址为网关的IP地址,代表其目的就是想获得网关的MAC地址。如果主机A已经知道网关的MAC地址，则略过此步。

2. 网关收到ARP广播包后同样会向主机A发回一个ARP应答包。当主机A收到的应答包中获得网关的MAC地址后，在主机A向主机B发送的原报文的目的MAC地址字段填上网关的MAC地址后发给网关。(目的IP还是主机B)

3. 如果网关的ARP表中已有主机B对应的MAC地址，则网关直接将在来自主机A的报文中的目的MAC地址字段填上主机B的MAC地址后转发给B。

4. 如果网关ARP表中没有主机B的MAC地址，网关会再次向主机B所在的网段发送ARP广播请求，此时目的IP地址为主机B的IP地址，当网关从收到来自主机B的应答报文中获得主机B的MAC地址后，就可以将主机A发来的报文重新再目的MAC地址字段填上主机B的MAC地址后发送给主机B。

### ARP协议格式
![ARP协议](https://img2018.cnblogs.com/blog/835745/201908/835745-20190812173139966-429857585.png)

- 以太网目的地址: 目的主机的硬件地址。目的地址全为1表示广播地址
- 以太网源地址：源主机的硬件地址
- 帧类型：ARP：0x0806、　RARP：0x8035
- 协议类型：IP类型：0x0800
- 硬件地址长度：对于以太网II来说，MAC地址作为硬件地址，因此该字段值为十六进制06
- 协议地址长度：对于IPv4来，IP地址长度位32个字节，因此该字段值为十六进制04
- 操作类型：ARP定义了两种操作：0x0001(请求)、0x0002(应答)
- 发送端以太网地址：对于以太网II来说，MAC地址作为硬件地址
- 发送端IP地址：对于IPv4来， 值为IPv4地址
- 目的以太网地址：对于以太网II来说，MAC地址作为硬件地址
- 目的IP地址：对于IPv4来， 值为IPv4地址

### 常见命令

#### arp
arp 命令用于显示和修改 IP 到 MAC 转换表

使用方法如下：
```
-a : 显示 arp 缓冲区的所有条目
```

#### arping

arping用来向局域网内的其它主机发送ARP请求，它可以用来测试局域网内的某个IP是否已被使用。

使用方法如下：
```
Usage: arping [-fqbDUAV] [-c count] [-w timeout] [-I device] [-s source] destination
  -f : quit on first reply （收到第一个响应包后退出)
  -q : be quiet（静默模式）
  -b : keep broadcasting, don't go unicast （发送以太网广播帧，arping在开始时使用广播地址，在收到回复后使用unicast单播地址）
  -D : duplicate address detection mode (重复地址探测模式，用来检测有没有IP地址冲突，如果没有IP冲突则返回0)
  -U : Unsolicited ARP mode, update your neighbours (无理由的/强制的 ARP模式去更新别的主机上的ARP CACHE列表中的本机的信息，不需要响应。)
  -A : ARP answer mode, update your neighbours (与-U参数类似，但是使用的是ARP REPLY包而非ARP REQUEST包)
  -V : print version and exit （打印版本并退出)
  -c count : how many packets to send (指定数量)
  -w timeout : how long to wait for a reply （等待时间）
  -I device : which ethernet device to use （指定网卡）
  -s source : source ip address （源IP）
  destination : ask for what ip address
```

### 参考文档
- https://www.cnblogs.com/onlycat/p/11340872.html#/cnblog/works/article/11340872