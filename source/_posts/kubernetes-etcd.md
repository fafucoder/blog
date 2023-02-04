---
title: etcd 获取 kubernetes 中的数据
date: 2020-12-14 20:35:44
tags:
- kubernetes
categories:
- kubernetes
---

### etcdctl 命令
etcd厂商提供了命令行客户端 etcdctl，可以使用客户端直接跟etcd交互

etcdctl使用方法:
![etcdctl 命令](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1glno5yzncaj312h0u0hdt.jpg)

### kubernetes etcd pod中执行命令
在kubernetes中执行etcdctl命令需要知道endpoint地址以及证书

#### 查看endpoint地址

可以通过kube-apiserver与etcd交互获取endpoint地址，通过 `ps -ef | grep kube-apiserver `能够获取到kube-apiserver的启动参数
![endpoint地址](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1glnobnpvozj31la0fc79m.jpg)

#### 获取证书
etcd证书一般在`/etc/kubernetes/pki`目录下，或者在 `/etc/ssl/etcd/ssl`目录下
![etcd证书](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1glnojh7e8sj31gy03e0t6.jpg)

### etcd命令操作

1. 设置etcdctl版本
export ETCDCTL_API=3

2. 执行etcdctl 命令(命令跟上面的一致)
```
# 获取 endpoint status

etcdctl --endpoints=https://192.168.0.171:2379 --cert=/etc/ssl/etcd/ssl/admin-node1.pem --key=/etc/ssl/etcd/ssl/admin-node1-key.pem  --cacert=/etc/ssl/etcd/ssl/ca.pem -w table endpoint status
+----------------------------+------------------+---------+---------+-----------+-----------+------------+
|          ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+----------------------------+------------------+---------+---------+-----------+-----------+------------+
| https://192.168.0.171:2379 | 7c4294e7b9a48ff3 |  3.3.12 |   19 MB |      true |         2 |    1916345 |
+----------------------------+------------------+---------+---------+-----------+-----------+------------+

# 获取endpoint health

etcdctl --endpoints=https://192.168.0.171:2379 --cert=/etc/ssl/etcd/ssl/admin-node1.pem --key=/etc/ssl/etcd/ssl/admin-node1-key.pem  --cacert=/etc/ssl/etcd/ssl/ca.pem -w table endpoint health
https://192.168.0.171:2379 is healthy: successfully committed proposal: took = 826.779µs

# 获取k8s注册的数据
etcdctl --endpoints=https://192.168.0.171:2379 --cert=/etc/ssl/etcd/ssl/admin-node1.pem --key=/etc/ssl/etcd/ssl/admin-node1-key.pem  --cacert=/etc/ssl/etcd/ssl/ca.pem get / --prefix --keys-only=true

/registry/apiextensions.k8s.io/customresourcedefinitions/alertmanagers.monitoring.coreos.com

/registry/apiextensions.k8s.io/customresourcedefinitions/applications.app.k8s.io

/registry/apiextensions.k8s.io/customresourcedefinitions/blockdeviceclaims.openebs.io

/registry/apiextensions.k8s.io/customresourcedefinitions/blockdevices.openebs.io

/registry/apiextensions.k8s.io/customresourcedefinitions/cdiconfigs.cdi.kubevirt.io

/registry/apiextensions.k8s.io/customresourcedefinitions/cdis.cdi.kubevirt.io

/registry/apiextensions.k8s.io/customresourcedefinitions/clusterconfigurations.installer.kubesphere.io

/registry/apiextensions.k8s.io/customresourcedefinitions/clusterpropagatedversions.core.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/clusters.cluster.kubesphere.io

/registry/apiextensions.k8s.io/customresourcedefinitions/dashboards.monitoring.kubesphere.io

/registry/apiextensions.k8s.io/customresourcedefinitions/datavolumes.cdi.kubevirt.io

/registry/apiextensions.k8s.io/customresourcedefinitions/devopsprojects.devops.kubesphere.io

/registry/apiextensions.k8s.io/customresourcedefinitions/dnsendpoints.multiclusterdns.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/domains.multiclusterdns.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/emailconfigs.notification.kubesphere.io

/registry/apiextensions.k8s.io/customresourcedefinitions/emailreceivers.notification.kubesphere.io

/registry/apiextensions.k8s.io/customresourcedefinitions/exporters.events.kubesphere.io

/registry/apiextensions.k8s.io/customresourcedefinitions/externalips.kubeovn.io

/registry/apiextensions.k8s.io/customresourcedefinitions/externalsubnets.kubeovn.io

/registry/apiextensions.k8s.io/customresourcedefinitions/federatedapplications.types.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/federatedclusterrolebindings.types.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/federatedclusterroles.types.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/federatedconfigmaps.types.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/federateddeployments.types.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/federatedglobalrolebindings.types.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/federatedglobalroles.types.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/federatedingresses.types.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/federatedjobs.types.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/federatedlimitranges.types.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/federatednamespaces.types.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/federatedpersistentvolumeclaims.types.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/federatedreplicasets.types.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/federatedsecrets.types.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/federatedserviceaccounts.types.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/federatedservices.types.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/federatedservicestatuses.core.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/federatedstatefulsets.types.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/federatedtypeconfigs.core.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/federatedusers.types.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/federatedworkspacerolebindings.types.kubefed.io

/registry/apiextensions.k8s.io/customresourcedefinitions/federatedworkspaceroles.types.kubefed.io

# 获取具体的数据(先拿到key,然后获取key的内容)
etcdctl --endpoints=https://192.168.0.171:2379 --cert=/etc/ssl/etcd/ssl/admin-node1.pem --key=/etc/ssl/etcd/ssl/admin-node1-key.pem  --cacert=/etc/ssl/etcd/ssl/ca.pem get /registry/kubeovn.io/subnets/ovn-default
/registry/kubeovn.io/subnets/ovn-default
{"apiVersion":"kubeovn.io/v1","kind":"Subnet","metadata":{"creationTimestamp":"2020-12-11T09:25:44Z","finalizers":["kube-ovn-controller"],"generation":1,"name":"ovn-default","uid":"89ac06b6-7ca8-4606-843e-6630eb818317"},"spec":{"cidrBlock":"10.233.64.0/18","default":true,"excludeIps":["10.233.64.1"],"gateway":"10.233.64.1","gatewayNode":"","gatewayType":"distributed","natOutgoing":true,"private":false,"protocol":"IPv4","provider":"ovn","underlayGateway":false},"status":{"activateGateway":"","availableIPs":16314,"conditions":[{"lastTransitionTime":"2020-12-11T09:25:50Z","lastUpdateTime":"2020-12-11T09:25:50Z","reason":"ResetLogicalSwitchAclSuccess","status":"True","type":"Validated"},{"lastTransitionTime":"2020-12-11T09:25:50Z","lastUpdateTime":"2020-12-11T09:25:50Z","reason":"ResetLogicalSwitchAclSuccess","status":"True","type":"Ready"}],"usingIPs":67}}

# curd
......

# etcd 数据备份
etcdctl --endpoints=https://192.168.0.171:2379 --cert=/etc/ssl/etcd/ssl/admin-node1.pem --key=/etc/ssl/etcd/ssl/admin-node1-key.pem  --cacert=/etc/ssl/etcd/ssl/ca.pem snapshot save /root/backup
Snapshot saved at /root/backup

# etcd 数据恢复
etcdctl --endpoints=https://192.168.0.171:2379 --cert=/etc/ssl/etcd/ssl/admin-node1.pem --key=/etc/ssl/etcd/ssl/admin-node1-key.pem  --cacert=/etc/ssl/etcd/ssl/ca.pem snapshot restore /root/backup
```

### 参考文档
- [etcd 及 etcd 在 k8s中的用法](https://blog.csdn.net/u013693952/article/details/106358339)
- [[译]走进Kubernetes集群的大脑：Etcd](https://juejin.cn/post/6844903997707386887)
- [etcd](https://feisky.gitbooks.io/kubernetes/content/components/etcd.html)
