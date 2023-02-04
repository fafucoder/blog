---
title: helm 包管理
date: 2020-04-13 12:41:47
tags:
- kubernetes
categories:
- kubernetes
---

### Helm 基础概念

Helm Chart: Chart 代表Helm包，它包含在 Kubernetes 集群内部运行应用程序，工具或服务所需的所有资源定义。可以把Helm Chart比作Apt dpkg或者yum rpm在linux中的等价位。

Helm Hub: 类似于docker hub 用于存放公共chart的地方

Helm Repository: 是用来存放和共享 charts 的地方(放置在本地)

Helm Release: Release 是运行在 Kubernetes 集群中的 chart 的实例。一个 chart 通常可以在同一个集群中安装多次。每一次安装都会创建一个新的 release。以 MySQL chart为例，如果你想在你的集群中运行两个数据库，你可以安装该chart两次。每一个数据库都会拥有它自己的 release 和 release name。

![helm 基础概念](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gpo89yf0pyj31740omdph.jpg)

### Helm 命令(详细参数查阅官方文档)

**helm search hub**: 从 [Artifact Hub](https://artifacthub.io/) 中查找并列出 helm charts。 Artifact Hub中存放了大量不同的仓库。

**helm search repo**: 从你添加（使用 helm repo add）到本地 helm 客户端中的仓库中进行查找。该命令基于本地数据进行搜索，无需连接互联网。

![helm search](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gpo8i9ww6zj31nw0bqwrk.jpg)

**helm repo list**: 列出所有本地已经添加的repo

**helm repo add**: 从repo hub中添加一个chart到本地repo

**helm repo remove**: 从本地repo中移除chart

**helm repo update**: 更新本地repo charts
![helm repo](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gpo8tclg3vj314s08wgqj.jpg)

**helm list**: 列出所有的release chart

**helm uninstall**: 删除一个release

**helm status**: 查看release状态(可以通过helm status 查看helm的输出信息，比如默认用户名密码可能通过helm status 来查看)

![helm status](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gpo9k7yks2j313k0cotae.jpg)

**helm install**: 安装一个chart

**helm upgrade**: 升级chart

**helm show values**: 列出chart 的自定义变量value

**helm get values**: 查看release 的values(也就是说可以查看哪些自定义的值作了修改)

**helm history**: 查看 release 的历史版本

![helm history](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gpo9ii92x6j31hm02wwf1.jpg)

**helm pull(fetch)**: 从包仓库(hub or repo)中检索包，并在本地下载它。

**helm template**: 本地渲染模板(也就是说把默认的信息转化成yaml)

![helm pull/template](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gpoa7qdls3j31th0u0ncl.jpg)

### 参考文档

- https://blog.csdn.net/qq_27234433/article/details/114338623    //helm文档
- https://helm.sh/zh/docs/helm/helm_repo/  //helm 官方文档