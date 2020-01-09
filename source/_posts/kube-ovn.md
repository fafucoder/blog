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
- http://maoqide.live/post/cloud/sample-controller //自定义资源(有图有真相)
- https://blog.csdn.net/boling_cavalry/article/details/88934063 //也是自定义资源的
- https://blog.openshift.com/kubernetes-deep-dive-code-generation-customresources/ //英文　自定义资源
- https://jimmysong.io/kubernetes-handbook/concepts/crd.html
- https://juejin.im/post/5daf8a8d6fb9a04e366a49a8