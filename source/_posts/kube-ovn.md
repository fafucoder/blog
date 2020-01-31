---
title: kube-ovn学习指南
date: 2020-01-09 22:31:08
tags: 
- kubernetes
- kube-ovn
---

### kube-ovn的网络架构
Kube-OVN 在安装时会配置两个内置子网：
- default 子网: 作为 Pod 分配 IP 使用的默认子网，默认 cidr 为 10.16.0.0/16
- node 子网: 作为 Node 和 Pod 之间进行网络通信的特殊子网, 默认 cidr 为 100.64.0.1/16

### kube-ovn的网络配置
```json
{
    "name":"kube-ovn",
    "cniVersion":"0.3.1",
    "plugins":[
        {
            "type":"kube-ovn",
            "server_socket":"/run/openvswitch/kube-ovn-daemon.sock"
        },
        {
            "type":"portmap",
            "capabilities":{
                "portMappings":true
            }
        }
    ]
}
```

### 参考网址
- http://weekly.dockerone.com/article/9076 //关于kube-ovn的网络架构和工作原理
- http://www.sohu.com/a/129910066_515888 //cni结构图
- https://blog.csdn.net/fy_long/article/details/86498976 //关于k8s网络
- https://segmentfault.com/a/1190000017182169 //关于CNI的内容
- https://juejin.im/post/5daf8a8d6fb9a04e366a49a8
- https://blog.csdn.net/Ay_Ly/article/details/89393281
- https://blog.csdn.net/rocson001/article/details/73163041
- https://www.sdnlab.com/19765.html
- https://neuvector.com/network-security/advanced-kubernetes-networking/