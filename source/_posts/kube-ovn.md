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


### 参考网址
- http://weekly.dockerone.com/article/9076  //关于kube-ovn的网络架构和工作原理
- https://github.com/alauda/kube-ovn/wiki   //kube-ovn如何使用
- https://neuvector.com/network-security/advanced-kubernetes-networking/  //其他网络插件流程