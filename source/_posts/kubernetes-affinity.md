---
title: kubernetes 亲和性和反亲和性调度
date: 2020-05-17 15:23:09
tags:
- kubernetes
categories:
- kubernetes
---

### 概述
kubernetes 提供将pod限制在指定的node上运行，或者指定更倾向于在某些特定的node上运行，有几种方式实现这个功能：
- NodeName: 最简单的节点选择方式，直接指定节点，跳过调度器。
- NodeSelector: 早期的简单控制方式，直接通过键—值对将 Pod 调度到具有特定 label 的 Node 上。
- NodeAffinity: NodeSelector 的升级版，支持更丰富的配置规则，使用更灵活。(NodeSelector 将被淘汰.)
- PodAffinity: 根据已在节点上运行的 Pod 标签来约束 Pod 可以调度到哪些节点，而不是根据 node label。

#### NodeName
nodeName 是 PodSpec 的一个字段，用于直接指定调度节点，并运行该 pod。调度器在工作时，实际选择的是 nodeName 为空的 pod 并进行调度然后再回填该 nodeName，所以直接指定 nodeName 实际是直接跳过了调度器。换句话说，指定 nodeName 的方式是优于其他节点选择方法。

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeName: kube-01
```

#### NodeSelector
nodeSelector 也是 PodSpec 中的一个字段，指定键—值对的映射。如果想要将 pod 运行到对应的 node 上，需要先给这些 node 打上 label，然后在 podSpec.NodeSelector 指定对应 node labels 即可。

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    type: gpu
```

### Affinity
affinity格式如下：

```
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      requiredDuringSchedulingIgnoredDuringExecution:
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      requiredDuringSchedulingIgnoredDuringExecution:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      requiredDuringSchedulingIgnoredDuringExecution:
```

Affinity 当前支持两种调度模式:

- requiredDuringSchedulingIgnoredDuringExecution: 一定要满足的条件，如果没有找到满足条件的节点，则 Pod 创建失败。所有也称为hard 模式。
- preferredDuringSchedulingIgnoredDuringExecution: 优先选择满足条件的节点，如果没有找到满足条件的节点，则在其他节点中择优创建 Pod。所有也称为 soft 模式。

其中IgnoredDuringExecution的意义就跟 nodeSelector 的实现一样，即使 node label 发生变更，也不会影响之前已经部署且又不满足 affinity rules 的 pods，这些 pods 还会继续在该 node 上运行。换句话说，亲和性选择节点仅在调度 Pod 时起作用。

nodeAffinity 当前支持的匹配符号包括：In、NotIn、Exists、DoesNotExists、Gt、Lt 。

#### NodeAffinity
```
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      # 必须选择 node label key 为 kubernetes.io/e2e-az-name,
      # value 为 e2e-az1 或 e2e-az2.
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      # 过滤掉上面的必选项后，再优先选择 node label key 为 another-node-label-key
      # value 为 another-node-label-value.
      preferredDuringSchedulingIgnoredDuringExecution:
      # 如果满足节点亲和，积分加权重(优选算法，会对 nodes 打分)
      # weight: 0 - 100
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:2.0
```

NodeAffinity 的结构体:
```
type NodeAffinity struct {
    RequiredDuringSchedulingIgnoredDuringExecution *NodeSelector
    PreferredDuringSchedulingIgnoredDuringExecution []PreferredSchedulingTerm
}

type NodeSelector struct {
    NodeSelectorTerms []NodeSelectorTerm
}

type NodeSelectorTerm struct {
    MatchExpressions []NodeSelectorRequirement
    MatchFields []NodeSelectorRequirement
}
```

配置相关的注意点：

- 如果 nodeSelector 和 nodeAffinity 两者都指定，那 node 需要两个条件都满足，pod 才能调度。
- 如果指定了多个 NodeSelectorTerms，那 node 只要满足其中一个条件，pod 就可以进行调度。
- 如果指定了多个 MatchExpressions，那必须要满足所有条件，才能将 pod 调度到该 node。


#### PodAffinity

跟NodeAffinity的调度方式类似

### 参考文档
- https://segmentfault.com/a/1190000018446833  //Kubernetes 亲和性调度
