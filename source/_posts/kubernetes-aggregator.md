---
title: kubernetes api聚合机制aggregation
date: 2021-01-29 19:11:37
tags:
- kubernetes
categories:
- kubernetes
---

### 概念

​	kubernetes的 Aggregated API是什么呢？API Aggregation 允许k8s的开发人员编写一个自己的服务，可以把这个服务注册到k8s的api里面，这样，就像k8s自己的api一样，你的服务只要运行在k8s集群里面，k8s 的Aggregate通过service名称就可以转发到你写的service里面去了。(另外一种扩展 Kubernetes API 的方法是使用[CustomResourceDefinition](https://feisky.gitbooks.io/kubernetes/content/concepts/customresourcedefinition.html), 编写的自定义CRD, k8s会自动的注册到apiservice上 )

这个设计理念：

1. 增加了api的扩展性，这样k8s的开发人员就可以编写自己的API服务器来公开他们想要的API。集群管理员应该能够使用这些服务，而不需要对核心库存储库进行任何更改。
2. 丰富了APIs，核心kubernetes团队阻止了很多新的API提案。通过允许开发人员将他们的API作为单独的服务器公开，并使集群管理员能够在不对核心库存储库进行任何更改的情况下使用它们，这样就无须社区繁杂的审查了
3. 开发分阶段实验性API的地方，新的API可以在单独的聚集服务器中开发，当它稳定之后，那么把它们封装起来安装到其他集群就很容易了。
4. 确保新API遵循kubernetes约定：如果没有这里提出的机制，社区成员可能会被迫推出自己的东西，这可能会或可能不遵循kubernetes约定。

### 原理

​	当APIAggregator接收到请求之后，如果发现对应的是一个service的请求，则会直接转发到对应的服务上否则委托给apiserver进行处理，apiserver中根据当前URL来选择对应的REST接口处理，如果未能找到对应的处理，则会交由CRD server处理， CRD server检测是否已经注册对应的CRD资源，如果注册就处理

![api aggregation](https://tva1.sinaimg.cn/large/008eGmZEly1gn86kq5rumj319y0c4my6.jpg)

​	当在集群中创建了对应的CRD资源的时候，k8s通过内部的controller来感知对应的CRD资源信息，然后为其创建对应的REST处理接口，这样后续接收到对应的资源就可以进行处理

![crd server](https://tva1.sinaimg.cn/large/008eGmZEly1gn86o0k0kqj314q0bwdh7.jpg)

​	APIAggreagtor中会通过informer 监听后端Service的变化，如果发现有新的服务，就会创建对应的代理转发，从而实现对应的服务注册

![api service](https://tva1.sinaimg.cn/large/008eGmZEly1gn86p5w6erj315g0b6t9x.jpg)

### 注册自定义service资源

```yaml
apiVersion: apiregistration.k8s.io/v1beta1
kind: APIService
metadata:
  name: v1beta1.custom.metrics.k8s.io
spec:
  service:
    name: custom-metrics-server
    namespace: custom-metrics
  group: custom.metrics.k8s.io
  version: v1beta1
  insecureSkipTLSVerify: true
  groupPriorityMinimum: 100
  versionPriority: 100
```

 在这个APIService中设置的API组名为custom.metrics.k8s.io，版本号为v1beta1，这两个字段将作API路径的子目录注册到API路径“/apis/”下。注册成功后，就能通过Master API路径“/apis/custom.metrics.k8s.io/v1beta1”访问自定义的API Server了, API聚合层会代理转发到后端服务custom-metrics-server.custom-metrics.svc上。

官方提供有一个 [自定义api service ](https://github.com/kubernetes/sample-apiserver) 样例, 通过官方样例可以创建一个自定义聚合api

### 注意点

1. service监听的端口必须是https,否则认证通过不了。(通过`kubectl get apiservice`能看到是否通过认证)![http报错](https://tva1.sinaimg.cn/large/008eGmZEly1gn85pg2d84j316u070788.jpg)

2. 请求的根路径需要应答请求，否则会报404错误（`kubectl get apiservice v1alpha3.demo.com.cn  -o yaml` 能拷看到状态）

   ![404状态](https://tva1.sinaimg.cn/large/008eGmZEly1gn85u1y817j316406atc7.jpg)
   
3. 请求的根路径的应答结果需要包含`kind`, `apiVersion`, `groupVersion`, `resources`：

   ```json
   {
   "kind": "APIResourceList",
   "apiVersion": "v1",
   "groupVersion": "metrics.k8s.io/v1beta1",
   "resources": []
   }
   ```

   

### 参考文档

- [图解kubernetes中api聚合机制的实现](https://studygolang.com/articles/26970)
- [解析kubernetes Aggregated API Servers](https://blog.csdn.net/u010278923/article/details/78890533)
- [Kubernetes之使用API聚合机制扩展API资源](https://blog.csdn.net/qq_31136839/article/details/100183026)
- [K8s Aggregation API体验](http://www.iceyao.com.cn/2019/06/10/k8s-aggregation-api%E4%BD%93%E9%AA%8C/)