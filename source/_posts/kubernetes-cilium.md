---
title: kubernetes-cilium
date: 2020-07-06 14:53:20
tags:
- kubernetes
categories:
- kubernetes
---

### cilium1.8 安装

#### 默认安装
```
helm install cilium ./cilium \
   --namespace kube-system \
   --set global.nodeinit.enabled=true \
   --set global.kubeProxyReplacement=partial \
   --set global.hostServices.enabled=false \
   --set global.externalIPs.enabled=true \
   --set global.nodePort.enabled=true \
   --set global.hostPort.enabled=true \
   --set global.pullPolicy=IfNotPresent \
   --set config.ipam=kubernetes \
   --set global.hubble.enabled=true \
   --set global.hubble.listenAddress=":10000" \
   --set global.hubble.relay.enabled=true \
   --set global.hubble.ui.enabled=true
```

#### cni-chaining安装方式
```
apiVersion: v1
data:
  cni-config: |-
    {
      "name": "generic-veth",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "kube-ovn",
          "log_level": "info",
          "datastore_type": "kubernetes",
          "mtu": 1400,
          "ipam": {
                "type": "kube-ovn",
                "server_socket": "/run/openvswitch/kube-ovn-daemon.sock"
          },
          "policy": {
              "type": "k8s"
          },
          "kubernetes": {
              "kubeconfig": "/etc/cni/net.d/01-kube-ovn.conflist"
          }
        },
        {
          "type": "cilium-cni"
        }
      ]
    }
kind: ConfigMap
metadata:
  name: cni-configuration
  namespace: kube-system


helm install cilium cilium/cilium --version 1.8.1 \
  --namespace=kube-system \
  --set global.cni.chainingMode=generic-veth \
  --set global.cni.customConf=true \
  --set global.cni.configMap=cni-configuration \
  --set global.tunnel=disabled \
  --set global.masquerade=false
  --set global.hubble.enabled=true \
  --set global.hubble.listenAddress=":10000" \
  --set global.hubble.relay.enabled=false \
  --set global.hubble.ui.enabled=true
```


#### 安装prometheus
```
helm install cilium cilium/cilium --version 1.8.1 \
  --namespace=kube-system \
  --set global.cni.chainingMode=generic-veth \
  --set global.cni.customConf=true \
  --set global.cni.configMap=cni-configuration \
  --set global.tunnel=disabled \
  --set global.masquerade=false \
  --set global.hubble.enabled=true \
  --set global.hubble.listenAddress=":4244" \
  --set global.hubble.relay.enabled=true \
  --set global.hubble.ui.enabled=true \
  --set global.kubeProxyReplacement=strict \
  --set global.k8sServiceHost=172.24.37.57 \
  --set global.k8sServicePort=6443 \
  --set global.prometheus.enabled=true \
  --set global.operatorPrometheus.enabled=true \
  --set global.hubble.enabled=true \
  --set global.hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,http}"
```

### 参考文档

#### 安装文档相关
- https://blog.csdn.net/F8qG7f9YD02Pe/article/details/79815702
- https://cilium.io/blog/2020/05/04/guest-blog-kubernetes-cilium/ 
- https://www.jianshu.com/p/090c3d32c2be

#### tc相关
- https://cloud.tencent.com/developer/article/1409664
- https://tonydeng.github.io/sdn-handbook/linux/tc.html

#### bpf相关
- https://blog.csdn.net/pwl999/article/details/82884882

#### xdp相关
- https://cloud.tencent.com/developer/article/1484793

#### 系列博客
- https://davidlovezoe.club/wordpress/