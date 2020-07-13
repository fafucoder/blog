---
title: iptables 学习记录
date: 2020-03-24 14:46:05
tags:
- linux
categories:
- linux
---

### 概念
iptables的底层实现是netfilter，整个流程图如下图所示。

![netfilter流程图](https://img30.360buyimg.com/ebookadmin/jfs/t1/106051/33/8821/143058/5e09a902Ea345c13e/109408b707aed1dd.jpg)


当网卡上收到一个包送达协议栈时，最先经过的netfilter钩子是PREROUTING，如果确实有用户埋了这个钩子函数，那么内核将在这里对数据包进行目的地址转换（DNAT）。不管在PREROUTING有没有做过DNAT，内核都会通过查本地路由表决定这个数据包是发送给本地进程还是发送给其他机器。如果是发送给其他机器（或其他network namespace），就相当于把本地当作路由器，就会经过netfilter的FORWARD钩子，用户可以在此处设置包过滤钩子函数，例如iptables的reject函数。所有马上要发到协议栈外的包都会经过POSTROUTING钩子，用户可以在这里埋下源地址转换（SNAT）或源地址伪装（Masquerade，简称Masq）的钩子函数。如果经过上面的路由决策，内核决定把包发给本地进程，就会经过INPUT钩子。本地进程收到数据包后，回程报文会先经过OUTPUT钩子，然后经过一次路由决策（例如，决定从机器的哪块网卡出去，下一跳地址是多少等），最后出协议栈的网络包同样会经过POSTROUTING钩子。

![数据流向](http://www.zsythink.net/wp-content/uploads/2017/02/021217_0051_2.png)

### iptables三板斧 table, rule, chain
iptables的工作流程如下图所示，
![iptables工作流程](https://img30.360buyimg.com/ebookadmin/jfs/t1/94052/37/9026/140242/5e09a8ffEb3fbbcf0/df2aae113a78cf60.jpg)

#### 五张链
- INPUT: 处理输入本地进程的数据包
- OUTPUT: 处理输入本地进程的输出数据包
- FORWORD: 处理转发到其他机器/network namespace的数据包
- PREROUTING: 目的地址转换DNAT
- POSTROUTING: 原地址转换SNAT

#### 五张表
- raw: 去除数据包连接追踪机制
- fileter: 控制到达某条链上的数据包是继续放行、直接丢弃（drop）或拒绝（reject）
- nat: 修改数据包的源和目的地址
- mangle: 修改数据包的IP头信息
- security: ....

#### 规则动作
- ACCEPT: 将数据包放行，进行完此处理动作后，将不再比对其它规则，直接跳往下一个规则链
- REJECT: 拦阻该数据包，并传送数据包通知对方
- DROP: 丢弃包不予处理，进行完此处理动作后，将不再比对其它规则，直接中断过滤程序
- REDIRECT: 将包重新导向到另一个端口（PNAT），进行完此处理动作后，将会继续比对其它规则。(iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-ports 8080)
- SNAT: 改写封包来源 IP 为某特定 IP 或 IP 范围，可以指定 port 对应的范围，进行完此处理动作后，将直接跳往下一个规则。(iptables -t nat -A POSTROUTING -p tcp-o eth0 -j SNAT --to-source 194.236.50.155-194.236.50.160:1024-32000)
- DNAT: 改写封包目的地 IP 为某特定 IP 或 IP 范围，可以指定 port 对应的范围，进行完此处理动作后，将会直接跳往下一个规炼。(iptables -t nat -A PREROUTING -p tcp -d 15.45.23.67 --dport 80 -j DNAT --to-destination 192.168.1.1-192.168.1.10:80-100)

#### 注意点
1. 表的优先级从高到低是：raw、mangle、nat、filter、security。
2. iptables支持用户自定义链， 不支持用户自定义表。
3. 不是每个链上都能挂表，关系如下图

![iptables链表关系](https://img30.360buyimg.com/ebookadmin/jfs/t1/89718/39/8857/189419/5e09a90aE78d193fe/95c4d2a993cb6a18.jpg)

4. iptables的每条链下面的规则处理顺序是从上到下逐条遍历的，除非中途碰到DROP，REJECT，RETURN这些内置动作。如果iptables规则前面是自定义链，则意味着这条规则的动作是JUMP，即跳到这条自定义链遍历其下的所有规则，然后跳回来遍历原来那条链后面的规则
![iptables遍历规则](https://img30.360buyimg.com/ebookadmin/jfs/t1/109967/35/2654/69696/5e09a902Edeaa7108/93b0056f310938a0.jpg)

5. 查询iptables时，默认是filter表

#### 数据经过的流程图
![iptables数据经过流程图](https://img30.360buyimg.com/ebookadmin/jfs/t1/90870/2/8881/182617/5e09a906E88527f1b/7a0a4cdc079421e3.jpg)

### iptables功能操作

#### 查询
iptables --line-numbers -t 表名 -n(不解析ip) -v(详细信息) -L 链名

#### 删除iptables
1. 先查看路由规则，iptables --line-numbers -nvL INPUT
2. 清除路由规则, iptables  -t tablename -D 链名 line-number (例如3)

### 参考文档
- http://www.zsythink.net/archives/1199 // iptables概念
- http://www.zsythink.net/archives/1493 // iptables查询
- https://blog.csdn.net/guochunyang/article/details/49867707 // iptables规则
