---
title: linux虚拟网络设备
date: 2020-06-14 19:56:41
tags:
- linux
categories:
- linux
---

### veth-pair概念

Linux 虚拟网络的背后都是由一个个的虚拟设备构成的。虚拟化技术没出现之前，计算机网络系统都只包含物理的网卡设备，通过网卡适配器，线缆介质，连接外部网络，构成庞大的 Internet。

veth-pair 是成对出现的一种虚拟网络设备，一端连接着协议栈，一端连接着彼此，数据从一端出，从另一端进。它的这个特性常常用来连接不同的虚拟网络组件，构建大规模的虚拟网络拓扑，比如连接 Linux Bridge、OVS、LXC 容器等。
![veth-pair](https://tva1.sinaimg.cn/large/008i3skNly1gwjmzkaaslj31g80hc40g.jpg)

### 创建veth-pair

通过iproute2包中的ip link跟ip netns可以创建一个完整的veth-pair

![veth-pair互通](https://tva1.sinaimg.cn/large/008i3skNly1gwjmympfjkj30ye0iamxx.jpg)

```
# 创建veth pair
sudo ip link add veth0 type veth peer name veth1

# 设置命名空间
sudo ip netns add ns1
sudo ip netns add ns2

# 把veth pair移到命名空间下面
sudo ip link set veth0 netns ns1
sudo ip link set veth1 netns ns2

# 查看命名空间网卡设备
sudo ip netns exec ns1 ip link show

# veth-pair设置ip地址
sudo ip netns exec ns1 ip address add 192.168.1.1/24 dev veth0
sudo ip netns exec ns2 ip address add 192.168.1.2/24 dev veth1

# 设置网卡为up状态
sudo ip netns exec ns1 ip link set veth0 up
sudo ip netns exec ns2 ip link set veth1 up
```

### 通过bridge连接veth-pair

Linux Bridge 相当于一台交换机，可以中转两个 namespace 的流量

![bridge veth-pair互通](https://tva1.sinaimg.cn/large/008i3skNly1gwjn4rstl3j30yg0imabf.jpg)

```
# 创建bridge
sudo ip link add br0 type bridge
sudo ip link set br0 up

# 创建veth-pair
sudo ip link add veth0 type veth peer name br0-veth0
sudo ip link add veth1 type veth peer name br0-veth1

# 把peer name 移到bridge中
sudo ip link set br0-veth0 master br0
sudo ip link set br0-veth1 master br0

# 设置peer name 为up
sudo ip link set br0-veth0 up
sudo ip link set br0-veth1 up

# 创建namespace
sudo ip netns add ns1
sudo ip netns add ns2

# 把peer name移到 namespace中
sudo ip link set veth0 netns ns1
sudo ip link set veth1 netns ns2

# veth-pair设置ip地址
sudo ip netns exec ns1 ip address add 192.168.1.1/24 dev veth0
sudo ip netns exec ns2 ip address add 192.168.1.2/24 dev veth1

# 设置网卡为up状态
sudo ip netns exec ns1 ip link set veth0 up
sudo ip netns exec ns2 ip link set veth1 up
```

经过以上流程，发现还是无法ping通， 原因待查

### ping原理

![ping原理](https://tva1.sinaimg.cn/large/008i3skNly1gwjn5b0u3nj30pe0ka759.jpg)

### 参考文档
- https://ctimbai.github.io/tags/veth-pair
