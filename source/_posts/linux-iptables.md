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
- SNAT: 改写封包来源 IP 为某特定 IP 或 IP 范围，可以指定 port 对应的范围，进行完此处理动作后，将直接跳往下一个规则链。通过`--to-source`指定SNAT转换后的地址(iptables -t nat -A POSTROUTING -p tcp-o eth0 -j SNAT --to-source 194.236.50.155-194.236.50.160:1024-32000)
- DNAT: 改写封包目的地 IP 为某特定 IP 或 IP 范围，可以指定 port 对应的范围，进行完此处理动作后，将会直接跳往下一个规则链。通过`--to-destination`指定DNAT转换后的地址(iptables -t nat -A PREROUTING -p tcp -d 15.45.23.67 --dport 80 -j DNAT --to-destination 192.168.1.1-192.168.1.10:80-100)
- MASQUERADE： 改写封包来源 IP 为防火墙 NIC IP，可以指定 port 对应的范围，进行完此处理动作后，直接跳往下一个规则链。这个功能与 SNAT 略有不同，当进行IP伪装时，不需指定要伪装成哪个IP，IP 会从网卡直接读取，当使用拨接连线时，IP 通常是由ISP公司的 DHCP 服务器指派的，这个时候 MASQUERADE 特别有用
- MARK: 将封包标上某个代号，以便提供作为后续过滤的条件判断依据，进行完此处理动作后，将会继续比对其它规则
- RETURN: 结束在目前规则链中的过滤程序，返回主规则链继续过滤
- QUEUE: 封包放入队列，交给其它程序处理
- MIRROR: 镜射封包，也就是将来源IP与目的地IP对调后，将封包送回，进行完此处理动作后，将会中断过滤程序
- LOG: 将封包相关讯息记录在`/var/log`中

#### 注意点
1. 表的优先级从高到低是：raw、mangle、nat、filter、security。
2. iptables支持用户自定义链， 不支持用户自定义表。
3. 不是每个链上都能挂表，关系如下图

![iptables链表关系](https://img30.360buyimg.com/ebookadmin/jfs/t1/89718/39/8857/189419/5e09a90aE78d193fe/95c4d2a993cb6a18.jpg)

4. iptables的每条链下面的规则处理顺序是从上到下逐条遍历的，除非中途碰到DROP，REJECT，RETURN这些内置动作。如果iptables规则前面是自定义链，则意味着这条规则的动作是JUMP，即跳到这条自定义链遍历其下的所有规则，然后跳回来遍历原来那条链后面的规则
![iptables遍历规则](https://img30.360buyimg.com/ebookadmin/jfs/t1/109967/35/2654/69696/5e09a902Edeaa7108/93b0056f310938a0.jpg)

5. 查询iptables时，默认是filter表

#### drop 跟reject区别
1. reject 动作会返回一个拒绝(终止)数据包(TCP FIN或UDP-ICMP-PORT-UNREACHABLE)，明确的拒绝对方的连接动作。 连接马上断开，Client会认为访问的主机不存在。
2. DROP动作只是简单的直接丢弃数据，并不反馈任何回应。需要Client等待超时。

#### 数据经过的流程图
![iptables数据经过流程图](https://img30.360buyimg.com/ebookadmin/jfs/t1/90870/2/8881/182617/5e09a906E88527f1b/7a0a4cdc079421e3.jpg)

### iptables功能操作

#### 命令格式
iptables command chain chain_name options

#### iptables 参数
```
Usage: iptables -[ACD] chain rule-specification [options]
       iptables -I chain [rulenum] rule-specification [options]
       iptables -R chain rulenum rule-specification [options]
       iptables -D chain rulenum [options]
       iptables -[LS] [chain [rulenum]] [options]
       iptables -[FZ] [chain] [options]
       iptables -[NX] chain
       iptables -E old-chain-name new-chain-name
       iptables -P chain target [options]
       iptables -h (print this help information)

Commands:
  --append  -A chain            			  Append to chain
  --check   -C chain            			  Check for the existence of a rule
  --delete  -D chain            			  Delete matching rule from chain
  --delete  -D chain rulenum    			  Delete rule rulenum (1 = first) from chain
  --insert  -I chain [rulenum]  			  Insert in chain as rulenum (default 1=first)
  --replace -R chain rulenum    			  Replace rule rulenum (1 = first) in chain
  --list    -L [chain [rulenum]] 			  List the rules in a chain or all chains
  --list-rules -S [chain [rulenum]] 		Print the rules in a chain or all chains
  --flush   -F [chain]          			  Delete all rules in  chain or all chains
  --zero    -Z [chain [rulenum]] 		 	  List Zero counters in chain or all chains
  --new     -N chain            			  Create a new user-defined chain
  --delete-chain -X [chain]         		Delete a user-defined chain
  --policy       -P chain target			 	Change policy on chain to target
  --rename-chain -E old-chain new-chain Change chain name, (moving any references)

Options:
    --ipv4          -4              				        Nothing (line is ignored by ip6tables-restore)
    --ipv6          -6              			 	        Error (line is ignored by iptables-restore)
[!] --protocol      -p proto        		            protocol: by number or name, eg. `tcp'
[!] --source        -s address[/mask][...]	        source specification
[!] --destination   -d address[/mask][...]	        destination specification
[!] --in-interface  -i input name[+]                network interface name ([+] for wildcard)
 	  --jump 	        -j target                       target for rule (may load target extension)
  	--goto          -g chain                        jump to chain with no return
    --match         -m match                        extended match (may load extension)
    --numeric       -n                              numeric output of addresses and ports
[!] --out-interface -o output name[+]               network interface name ([+] for wildcard)
    --table         -t table                        table to manipulate (default: `filter')
    --verbose       -v                              verbose mode
    --wait          -w [seconds]                    maximum wait to acquire xtables lock before give up
    --wait-interval -W [usecs]                      wait time to try to acquire xtables lock, default is 1 second
    --line-numbers                                  print line numbers when listing
    --exact         -x                              expand numbers (display exact values)
[!] --fragment      -f                              match second or further fragments only
    --modprobe=<command>                            try to insert modules using this command
    --set-counters PKTS BYTES                       set the counter during insert/append
[!] --version       -V                              print package version.
```
#### 查询
iptables --line-numbers -t 表名 -n(不解析ip) -v(详细信息) -L 链名

#### 删除iptables
1. 先查看路由规则，iptables --line-numbers -nvL INPUT
2. 清除路由规则, iptables  -t tablename -D 链名 line-number (例如3)

### iptables-save

iptables-save命令可以将内核中的当前iptables配置导出到标准输出中，可以通过IO重定向功能输出到文件，例如`iptables-save > hello.txt` 通过使用iptables-save能更直观的看懂iptables路由规则

#### 参数
-t(--table): 指定iptables中的表
-c(--counters): 当前的数据包计数器和字节计数器信息

### 参考文档
- http://www.zsythink.net/archives/1199 // iptables概念
- http://www.zsythink.net/archives/1493 // iptables查询
- https://blog.csdn.net/guochunyang/article/details/49867707 // iptables规则
- https://blog.csdn.net/u011537073/article/details/82685586