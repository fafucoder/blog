---
title: kubernetes api server接口交互
date: 2020-02-04 16:53:12
tags:
- kubernetes
categories:
- kubernetes
---

### API Server功能

k8s API Server提供了k8s各类资源对象（pod,RC,Service等）的增删改查及watch等HTTP Rest接口，是整个系统的数据总线和数据中心。

kubernetes API Server的功能：

- 提供了集群管理的REST API接口(包括认证授权、数据校验以及集群状态变更)；
- 提供其他模块之间的数据交互和通信的枢纽（其他模块通过API Server查询或修改数据，只有API Server才直接操作etcd）;
- 是资源配额控制的入口， 拥有完备的集群安全机制.

### API Server原理
![api server原理](http://res.cloudinary.com/dqxtn0ick/image/upload/v1510579017/article/kubernetes/core/kube-apiserver.png)

![api 功能图](https://feisky.gitbooks.io/kubernetes/components/assets/API-server-space.png)

- api: k8s的内置的资源类型
- apis:  用户自定义的crd资源类型
- healthz: 健康检查
- logs: 日志

### API Server 接口访问

#### 本地端口
1. 该端口用于接收HTTP请求；
2. 该端口默认值为8080，可以通过API Server的启动参数“--insecure-port”的值来修改默认值；
3. 默认的IP地址为“localhost”，可以通过启动参数“--insecure-bind-address”的值来修改该IP地址；
4. 非认证或授权的HTTP请求通过该端口访问API Server。

#### 安全端口
1. 该端口默认值为6443，可通过启动参数“--secure-port”的值来修改默认值；
2. 默认IP地址为非本地（Non-Localhost）网络端口，通过启动参数“--bind-address”设置该值；
3. 该端口用于接收HTTPS请求；
4. 用于基于Tocken文件或客户端证书及HTTP Base的认证；
5. 用于基于策略的授权；
6. 默认不启动HTTPS安全访问控制。

#### 访问方式
1. 通过 kube-proxy 访问集群： `kubectl proxy --port=8080`
2. 使用token (参考里面的第二个地址)
3. 使用编程方式调用(client-go调用)
4. 使用kubectl (kubectl get --raw /api/v1/namespaces)

#### 查看API

##### 获取所有的api

1. `kubectl api-versions`
2. `curl localhost:8080/api`

##### pods api
```bash
api/v1/pods
api/v1/nodes/{name}/proxy/pods
```

#### node api
```bash
api/v1/nodes/{name}/proxy/pods    #列出指定节点内所有Pod的信息
api/v1/nodes/{name}/proxy/stats   #列出指定节点内物理资源的统计信息
api/v1/nodes/{name}/proxy/spec    #列出指定节点的概要信息
```

### 参考文档
- https://www.huweihuang.com/kubernetes-notes/principle/kubernetes-core-principle-api-server.html //API Server简介
- https://k8smeetup.github.io/docs/tasks/administer-cluster/access-cluster-api/  //通过 Kubernetes API 访问集群
- https://juejin.im/post/5c934e5a5188252d7c216981  //Kubernetes源码分析之kube-apiserver
- https://feisky.gitbooks.io/kubernetes/components/apiserver.html   //非常详细，推荐阅读