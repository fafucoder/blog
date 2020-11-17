---
title: linux netstat命令
date: 2020-11-17 20:19:13
tags:
- linux
categories:
- linux
---

### 简介

netstat 是控制台命令，是一个监控tcp/ip网络有用的工具， n用于显示各种网络相关信息， 如网络连接， 路由表，接口状态,  多播成员， masquerade连接等。一般用于检测本机端口的网络连接情况。

### 常用命令参数

| 参数 | 说明                                |
| ---- | ------------------------------------- |
| -a   | 列出所有的连接                 |
| -t   | 列出所有的tcp连接              |
| -u   | 列出所有的udp连接              |
| -n   | 禁用域名解析                    |
| -l   | 仅列出有在 Listen (监听) 的服务状态。 |
| -p   | 显示PID                             |
| -c   | 持续输出，定时执行netstat命令 |

### 检查网卡状态

netstat 能够列举出所有的网卡，或者看网卡详情，与此功能相同的命令为`ip 或者 ethtool`。

```bash
-I, --interfaces=<Iface> 展示网卡详情
-i, --interfaces         列出所有的网卡
```

通过netstat -I 参数可以统计网卡的包统计情况，可用于网卡丢包排查

```bash
[root@xxx ~]# netstat -Ieth0
Iface             MTU    RX-OK RX-ERR RX-DRP RX-OVR    TX-OK TX-ERR TX-DRP TX-OVR Flg
eth0             1500  4976182      0      0 0       2752891      0      0      0 BMRU
```

### 统计包信息

通过`netstat -s` 显示网络的统计信息，包括接收到的ip/icmp/tcp/udp包数，转发的ip/icmp/tcp/udp包数等

```bash
[root@node1 ~]# netstat -s
Ip:
    321020293 total packets received
    4096798 forwarded
    17 with unknown protocol
    0 incoming packets discarded
    316923403 incoming packets delivered
    326072495 requests sent out
    3651 outgoing packets dropped
    108 dropped because of missing route
Icmp:
    315961 ICMP messages received
    629 input ICMP message failed.
    ICMP input histogram:
        destination unreachable: 1208
        timeout in transit: 7
        echo requests: 296605
        echo replies: 18140
        timestamp request: 1
    318614 ICMP messages sent
    0 ICMP messages failed
    ICMP output histogram:
        destination unreachable: 3869
        echo request: 18139
        echo replies: 296605
        timestamp replies: 1
  ......
```

### 列举路由表

netstat能列出当前主机的路由表，曾几何时就知道查看路由表只有`ip route 跟 route` ,不曾想netstat也能列出路由表

`netstat -r` 用于列出当前主机的路由表

```bash
[root@xxx ~]# netstat -r
Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
default         gateway         0.0.0.0         UG        0 0          0 eth0
link-local      0.0.0.0         255.255.0.0     U         0 0          0 eth0
192.168.0.0     0.0.0.0         255.255.255.0   U         0 0          0 eth0
```

### 参考文档

- [netstat命令详解](https://www.cnblogs.com/ggjucheng/archive/2012/01/08/2316661.HTML)
- [Netstat 的10个基本用法](https://blog.csdn.net/dviewer/article/details/51340587)