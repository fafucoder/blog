---
title: kubernetes 资源分配之 request 和 limit
date: 2020-11-05 19:46:27
tags: 
- kubernetes
categories:
- kubernetes
---

#### request

- 容器使用的最小资源需求， 作为容器调度时资源分配的判断依据
- 只有当前节点上可分配的资源 >= request 才允许将容器调度到该节点
- request参数不限制容器的最大可用资源

#### limit

- 容器能使用资源的最大值
- 设置为0表示对使用的资源不做限制，可无限的使用

#### request 跟 limit 的关系

request能保证pod有足够的资源来运行, 而limit则是防止某个pod无限制的使用资源, 导致其他pod崩溃. 两者的关系必须满足:

``````mathematica
0 <= request <= limit <= Infinity
``````

### kubernetes Qos类型

container `request`值表示容器保证可被分配到资源。container`limit`表示容器可允许使用的最大资源。Pod级别的`request`和`limit`是其所有容器的request和limit之和。

在机器资源超卖的情况下（limit的总量大于机器的资源容量），即CPU或内存耗尽，将不得不杀死(驱逐)部分不重要的容器。kubernetes通过kubelet来进行回收策略控制，保证节点上pod在节点资源比较小时可以稳定运行。

**对于CPU，如果pod中服务使用CPU超过设置的limits，pod不会被kill掉但会被限制。如果没有设置limits，pod可以使用全部空闲的cpu资源。**

**对于内存，当一个pod使用内存超过了设置的limits，pod中container的进程会被kernel因OOM kill掉。当container因为OOM被kill掉时，系统倾向于在其原所在的机器上重启该container或本机或其他重新创建一个pod。**

在Kubernetes中，pod的QoS级别包括：*Guaranteed*, *Burstable*与 *Best-Effort* ，三个级别的优先级依次递减。

#### Guaranteed

所有的容器的`limit`值和`request`值被配置且两者相等（如果只配置limit没有request，则request取值于limit）。

```yaml
# 示例1
containers:
  name: foo
    resources:
      limits:
        cpu: 10m
        memory: 1Gi
  name: bar
    resources:
      limits:
        cpu: 100m
        memory: 100Mi
# 示例2
containers:
  name: foo
    resources:
      limits:
        cpu: 10m
        memory: 1Gi
      requests:
        cpu: 10m
        memory: 1Gi

  name: bar
    resources:
      limits:
        cpu: 100m
        memory: 100Mi
      requests:
        cpu: 100m
        memory: 100Mi
```

#### Burstable

如果一个或多个容器的limit和request值被配置且两者不相等。

```yaml
# 示例1
containers:
  name: foo
    resources:
      limits:
        cpu: 10m
        memory: 1Gi
      requests:
        cpu: 10m
        memory: 1Gi

  name: bar

# 示例2
containers:
  name: foo
    resources:
      limits:
        memory: 1Gi

  name: bar
    resources:
      limits:
        cpu: 100m

# 示例3
containers:
  name: foo
    resources:
      requests:
        cpu: 10m
        memory: 1Gi

  name: bar
```

### Best-Effort

所有的容器的`limit`和`request`值都没有配置。

```yaml
containers:
  name: foo
    resources:
  name: bar
    resources:
```

### 参考文档

- https://cloud.tencent.com/developer/article/1004976
- https://www.cnblogs.com/sunsky303/p/12626608.html
- https://www.huweihuang.com/kubernetes-notes/resource/quality-of-service.html