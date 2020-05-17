---
title: kubernetes taint(污点)跟toleration(容忍)
date: 2020-05-17 10:35:32
tags:
- kubernetes
categories:
- kubernetes
---

### taint命令

#### 添加taint
```
kubectl taint node [node] key=value:[ NoSchedule | PreferNoSchedule | NoExecute ]

例如:
➜  ~ kubectl taint node central key1=value1:NoSchedule  
➜  ~ kubectl taint node central key2=value2:NoExecute 
➜  ~ kubectl taint node central key3=value3:PreferNoSchedule 
```

其中：
- Noschedule: 一定不能被调度
- PreferNoSchedule: 尽量不要被调度
- NoExecute: 不仅不会调度，还会驱逐Node上已有的pod

#### 查看taint
```
kubectl describe node node 或者  kubectl get node nodeName -o go-template={{.spec.taints}}

例如
➜  ~ kubectl get node central -o go-template={{.spec.taints}}
[map[effect:PreferNoSchedule key:key3 value:value3] map[effect:NoExecute key:key2 value:value2] map[effect:NoSchedule key:key1 value:value1]]
```

#### 删除taint
```
kubectl taint node nodename key:[NoSchedule | PreferNoSchedule | NoExecute]- 或者 kubectl taint node nodename key- //如果key是唯一的可以不指定调度方式

例如
➜  ~ kubectl taint node central key2:NoExecute-
➜  ~ kubectl taint node central key1-                 
```

#### kubernetes master节点
kubernetes 的master节点默认添加了`node-role.kubernetes.io/master=true`的NoSchedule taints, 因此所有的pod都不会调度到master节点上，如果要去除这个taint可以执行如下命令, 完成后master就可以按正常的node节点被调度了
```
kubectl taint node nodename node-role.kubernetes.io/master-
```

如果需要为master添加回taint可以执行以下命令：
```
kubectl taint nodes nodeName node-role.kubernetes.io/master=:NoSchedule
```

#### taint test pod
1. 为node 设置taint
```
kubectl taint node central key1=value1:NoSchedule

```

2. 创建taint pod
```
apiVersion: v1
kind: Pod
metadata:
 name: taint-test
 labels:
  key1: value1
spec:
 containers:
 - name: nginx
   image: nginx
   ports:
   - containerPort: 80
```

3. 结果
```
➜  yamls kubectl get pod -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP          NODE      NOMINATED NODE   READINESS GATES
taint-test               0/1     Pending   0          4m16s   <none>      <none>    <none>           <none>

```

### taint跟toleration含义

NodeAffinity节点亲和性，是Pod上定义的一种属性，使Pod能够按我们的要求调度到某个Node上，而Taints则恰恰相反，它可以让Node拒绝运行Pod，甚至驱逐Pod。

Taints(污点)是Node的一个属性，设置了Taints(污点)后，因为有了污点，所以Kubernetes是不会将Pod调度到这个Node上的，
于是Kubernetes就给Pod设置了个属性Tolerations(容忍)，只要Pod能够容忍Node上的污点，那么Kubernetes就会忽略Node上的污点，就能够(不是必须)把Pod调度过去。因此 Taints(污点)通常与Tolerations(容忍)配合使用。

`kubectl taint node nodename key=value:NoSchedule`  表示此节点已被key=value污染，Pod调度不允许（PodToleratesNodeTaints策略）或尽量不（TaintTolerationPriority策略）调度到此节点，除非是能够容忍（Tolerations）key=value污点的Pod。

### taint字段
```
type Taint struct {
    Key string
    Value string
    Effect TaintEffect
    // add taint 的时间点
    // 只有 Effect = NoExecute, 该值才会被 nodeController 写入
    TimeAdded *metav1.Time
}

type Toleration struct {
    Key string
    Operator TolerationOperator
    Value string
    Effect TaintEffect
    // 容忍时间
    TolerationSeconds *int64
}

type TaintEffect string

const (
    TaintEffectNoSchedule TaintEffect = "NoSchedule"
    TaintEffectPreferNoSchedule TaintEffect = "PreferNoSchedule"
    TaintEffectNoExecute TaintEffect = "NoExecute"
)

type TolerationOperator string

const (
    TolerationOpExists TolerationOperator = "Exists"
    TolerationOpEqual  TolerationOperator = "Equal"
)
```

#### taints注意事项
- 如果至少有一个 effect == NoSchedule 的 taint 没有被 pod toleration，那么 pod 不会被调度到该节点上。
- 如果所有 effect == NoSchedule 的 taints 都被 pod toleration，但是至少有一个 effect == PreferNoSchedule 没有被 pod toleration，那么 k8s 将努力尝试不把 pod 调度到该节点上。
- 如果至少有一个 effect == NoExecute 的 taint 没有被 pod toleration，那么不仅这个 pod 不会被调度到该节点，甚至这个节点上已经运行但是也没有设置容忍该污点的 pods，都将被驱逐

### tolerations

toleration的格式如下:
```
  tolerations:
  - effect:
    key:
    operator: 
    tolerationSeconds:
    value
```

其中operator包含以下两类:
- Exists: 这个配置下，不需要指定 value。
- Equal: 需要配置 value 值。(operator 的默认值)

有几个特殊情况:

- key 为空并且 operator 等于 Exists，表示匹配了所有的 keys，values 和 effects。换句话说就是容忍了所有的 taints。
```
tolerations:
- operator: "Exists"
```

- effect 为空，则表示匹配所有的 effects（NoSchedule、PreferNoSchedule、NoExecute）
```
tolerations:
- key: "key"
  operator: "Exists"

```

- TolerationSeconds与 effect 为 NoExecute 配套使用。用来指定在 node 添加了 effect = NoExecute 的 taint 后，能容忍该 taint 的 pods 可停留在 node 上的时间。
```
tolerations: 
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600

```

#### tolerations test pod
1. 创建toletation pod
```
apiVersion: v1
kind: Pod
metadata:
 name: toleration-test
 labels:
  key1: value1
spec:
 containers:
 - name: nginx
   image: nginx
   ports:
   - containerPort: 80
 tolerations:
 - key: "key1"
   value: "value1"
   operator: "Equal"
   effect: "NoSchedule"
```

2. 结果：
```
➜  yamls kubectl get pod -o wide
NAME                     READY   STATUS    RESTARTS   AGE    IP          NODE      NOMINATED NODE   READINESS GATES
taint-test               0/1     Pending   0          21m    <none>      <none>    <none>           <none>
toleration-test          1/1     Running   0          111s   10.42.0.9   central   <none>           <none>
```

### 参考文档 
- https://segmentfault.com/a/1190000018446858?utm_source=tag-newest //深入理解taint
- https://kubernetes.io/zh/docs/concepts/configuration/taint-and-toleration //官方理解
- https://www.jianshu.com/p/09356acd6991 //kubernetes taints 配置