---
title: linux ssh命令知多少
date: 2020-08-04 20:49:26
tags:
- linux
categories:
- linux
---

### 基础知识

##### 正向代理

所谓正向代理就是无法直接访问目标服务，只能借助中间机器来访问目标服务。如下图所示：hostA无法直接访问目标服务(例如在内网环境，无法直接访问外部服务)，但是能够访问hostB且hostB能够访问目标服务，因此hostA去访问hostB，然后由hostB去请求目标服务。

![正向代理](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp4kdcrt5fj31a60juq8m.jpg)

##### 反向代理

反向代理跟正向代理相反，正向代理解决的是怎么出去问题，反向代理解决的是怎么让外部访问你的服务问题，例如hostA有个服务需要对外暴露，但是外部网络无法直接访问,此时hostB能访问到hostA，外部网络能访问hostB，因此外部网络通过访问hostB,hostB再把数据代理到HostA

![反向代理](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gp4kt6oa33j31a60g4af8.jpg)



### ssh 代理功能

ssh 命令除了登陆外还有三种代理功能：

- **正向代理（-L）**：相当于 iptable 的 port forwarding
- **反向代理（-R）**：相当于 frp 或者 ngrok
- **socks5 代理（-D）**：相当于 ss/ssr

#### ssh 正向代理

```shell
ssh -L [客户端IP或省略]:[客户端端口]:[服务器侧能访问的IP]:[服务器侧能访问的IP的端口] [登陆服务器的用户名@服务器IP] -p [服务器ssh服务端口（默认22）]

```

客户端IP可以省略，省略的话就是127.0.0.1。服务器IP都可以用域名来代替。

举例说明： 你的IP是192.168.1.2，你可以ssh到某台服务器8.8.8.8，8.8.8.8可以访问8.8.4.4，你内网里还有一台机器可以访问你。 如果你想让内网里的另外一台电脑访问8.8.4.4的80端口的http服务，那么可以执行：

```shell
ssh -L 192.168.1.2:80:8.8.4.4:80 test@8.8.8.8
```

这时内网里的另外一台机器可以通过浏览器中输入`http://192.168.1.2:80`查看8.8.4.4的网页

#### ssh 反向代理

````shell
ssh -R [服务器IP或省略]:[服务器端口]:[客户端侧能访问的IP]:[客户端侧能访问的IP的端口] [登陆服务器的用户名@服务器IP] -p [服务器ssh服务端口（默认22）]
````

服务器IP如果省略，则默认为127.0.0.1，只有服务器自身可以访问。指定服务器外网IP的话，任何人都可以通过[服务器IP:端口]来访问服务。

举例说明： 你的IP是192.168.1.2，你可以ssh到外网某台服务器8.8.8.8，你内网里有一台机器192.168.1.3。 如果你想让外网所有的能访问8.8.8.8的IP都能访问192.168.1.3的http服务，那么可以执行：

```shell
ssh -R 8.8.8.8:80:192.168.1.3:80 test@8.8.8.8
```

这时外网任何一台可以访问8.8.8.8的机器都可以通过80端口访问到内网192.168.1.3机器的80端口。

### 参考文档

- https://cloud.tencent.com/developer/article/1114279
- https://blog.csdn.net/sch0120/article/details/73770386
- https://kanda.me/2019/07/01/ssh-over-http-or-socks/
- [ssh命令的三种代理功能](https://www.cnblogs.com/cangqinglang/p/12732661.html)

