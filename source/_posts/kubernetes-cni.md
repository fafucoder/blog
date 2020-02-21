---
title: kubernetes网络接口插件cni
date: 2020-02-02 19:08:23
tags:
- kubernetes
- cni
categories:
- kubernetes
---

### CNI配置

cni的默认配置目录在`/etc/cni/cni.d/`

cni的默认可执行目录在 `/opt/cni/bin`

可以在kubelet中修改默认的路径, 在/etc/systemd/system/kubelet.service.d/10-kubeadm.conf这个文件中的`EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env`里有关于cni的配置， 通过修改/kubeadm-flags.env可以修改cni的配置


### CNI Plugins

CNI的插件库主要有:

- [cni interfaces](https://github.com/containernetworking/cni)
- [cni plugins](https://github.com/containernetworking/plugins)

#### [CNI Interfaces](https://github.com/containernetworking/cni)
CNI项目
- libcni： 实现 CNI runtime 的 SDK，比如 kubernetes 里面的 NetworkPlugin 部分就使用了 libcni 来调用 CNI plugins. 这里面也有一些 interface，容易混淆，这个 interface 是对于 runtime 而言的，并不是对 plugin 的约束，比如 AddNetworkList, runtime 调用这个方法后，会按顺序执行对应  plugin 的 add 命令。 libcni 里面的 config caching：解决的是如果 ADD 之后配置变化了，如何 DEL 的问题.
- skel: 实现 CNI plugin 的骨架代码
- cnitool: 模拟 runtime 执行比如 libcni 的 AddNetworkList，触发执行 cni plugins

CNI Plugin对runtime的假设
- Container runtime 先为 container 创建 network 再调用 plugins.
- Container runtime 决定 container 属于哪个网络，继而决定执行哪个 CNI plugin.
- Network configuration 为 JSON 格式，包含一些必选可选参数.
- 创建时 container runtime 串行添加 container 到各个 network (ADD).
- 销毁时 container runtime 逆序将 container 从各个 network 移除 (DEL).
- 单个 container 的 ADD/DEL 操作串行，但是多个 container 之间可以并发.
- ADD 之后必有 DEL，多次 DEL 操作幂等.
- ADD 操作不会执行两次（对于同样的 KEY-network name, CNI_CONTAINERID, CNI_IFNAME）

CNI的运行流程， [具体方法](https://github.com/containernetworking/cni/blob/master/SPEC.md)
1. 基本操作: ADD, DEL, CHECK and VERSION
2. Plugins 是二进制，当需要 network 操作时，runtime 执行二进制对应 命令
3. 通过 stdin 向 plugin 输入 JSON 格式的配置文件，以及其他 container 相关的信息 比如：Container ID, Network namespace path, Network configuration, Extra arguments, Name of the interface inside the container
4. 通过 stdout 返回结果, 返回包括Interfaces list, IP configuration assigned to each interface, DNS information

下面进行具体的代码分析：

CNI接口实现主要有AddNetwork, CheckNetwork, DelNetwork
```golang
// github.com/containernetworking/cni/libcni/api.go

type CNI interface {
	AddNetworkList(ctx context.Context, net *NetworkConfigList, rt *RuntimeConf) (types.Result, error)
	CheckNetworkList(ctx context.Context, net *NetworkConfigList, rt *RuntimeConf) error
	DelNetworkList(ctx context.Context, net *NetworkConfigList, rt *RuntimeConf) error
	GetNetworkListCachedResult(net *NetworkConfigList, rt *RuntimeConf) (types.Result, error)
	GetNetworkListCachedConfig(net *NetworkConfigList, rt *RuntimeConf) ([]byte, *RuntimeConf, error)

	AddNetwork(ctx context.Context, net *NetworkConfig, rt *RuntimeConf) (types.Result, error)
	CheckNetwork(ctx context.Context, net *NetworkConfig, rt *RuntimeConf) error
	DelNetwork(ctx context.Context, net *NetworkConfig, rt *RuntimeConf) error
	GetNetworkCachedResult(net *NetworkConfig, rt *RuntimeConf) (types.Result, error)
	GetNetworkCachedConfig(net *NetworkConfig, rt *RuntimeConf) ([]byte, *RuntimeConf, error)

	ValidateNetworkList(ctx context.Context, net *NetworkConfigList) ([]string, error)
	ValidateNetwork(ctx context.Context, net *NetworkConfig) ([]string, error)
}

```

CNI结构体
```golang

type NetworkConfig struct {
	Network *types.NetConf   //NetConf describes a network.
	Bytes   []byte
}

type CNIConfig struct {
	Path     []string
	exec     invoke.Exec
	cacheDir string
}

type NetConf struct {
	CNIVersion string `json:"cniVersion,omitempty"`

	Name         string          `json:"name,omitempty"`
	Type         string          `json:"type,omitempty"`
	Capabilities map[string]bool `json:"capabilities,omitempty"`
	IPAM         IPAM            `json:"ipam,omitempty"`
	DNS          DNS             `json:"dns"`

	RawPrevResult map[string]interface{} `json:"prevResult,omitempty"`
	PrevResult    Result                 `json:"-"`
}

type RuntimeConf struct {
	ContainerID string
	NetNS       string
	IfName      string
	Args        [][2]string

	CapabilityArgs map[string]interface{}

	CacheDir string
}
```

AddNetwork操作(CheckNetwork和Deletework同理)
```golang
//直接调用addwork,结果缓存
func (c *CNIConfig) AddNetwork(ctx context.Context, net *NetworkConfig, rt *RuntimeConf) (types.Result, error) {
	result, err := c.addNetwork(ctx, net.Network.Name, net.Network.CNIVersion, net, nil, rt)
	if err != nil {
		return nil, err
	}

	if err = c.cacheAdd(result, net.Bytes, net.Network.Name, rt); err != nil {
		return nil, fmt.Errorf("failed to set network %q cached result: %v", net.Network.Name, err)
	}

	return result, nil
}


func (c *CNIConfig) addNetwork(ctx context.Context, name, cniVersion string, net *NetworkConfig, prevResult types.Result, rt *RuntimeConf) (types.Result, error) {
	// 一系列验证.....

	//生成networkconfig
	newConf, err := buildOneConfig(name, cniVersion, net, prevResult, rt)
	if err != nil {
		return nil, err
	}

	
	//执行pluginPath下的cni插件， netConf.Bytes写入stdin
	return invoke.ExecPluginWithResult(ctx, pluginPath, newConf.Bytes, c.args("ADD", rt), c.exec)
}
```

1. CNIConfig: 主要数据成员是plugin的路径，并实现了CNI接口
2. NetworkConfig和NetworkConfigList： 包括在/etc/cni/net.d下面的配置
3. RuntimeConf: 定义了runtimeConf配置
4. AddNetwork(): 先从pluginPath获得plugin的binary，然后injectRuntimeConfig()将网络配置注入到networkconfig中，并作为最后plugin执行的输入，然后还会将network的操作（ADD或者DEL）以及RuntimeConf，作为plugin执行时的变量


#### [CNI Plugins](https://github.com/containernetworking/plugins/tree/master/plugins)

cni plugins库中提供的插件类型有main, ipam, meta

main是实现了某种特定网络功能的插件; meta本身并不会提供具体的网络功能，它会调用其他插件，或者单纯是为了测试；ipam是分配IP地址的插件, ipam并不提供某种网络功能，只是为了灵活性把它单独抽象出来，这样不同的网络插件可以根据需求选择ipam，或者实现自己的ipam。

> main
>- loopback：这个插件很简单，负责生成 lo 网卡，并配置上 127.0.0.1/8 地址
>- bridge：和 docker 默认的网络模型很像，把所有的容器连接到虚拟交换机上
>- macvlan：使用 macvlan 技术，从某个物理网卡虚拟出多个虚拟网卡，它们有独立的 ip 和 mac 地址
>- ipvlan：和 macvlan 类似，区别是虚拟网卡有着相同的 mac 地址
>- ptp：通过 veth pair 在容器和主机之间建立通道

> meta
>- flannel：结合 bridge 插件使用，根据 flannel 分配的网段信息，调用 bridge 插件，保证多主机情况下容器
>- tuning：调整现有接口的sysctl参数
>- portmap：一个基于iptables的portmapping插件。将端口从主机的地址空间映射到容器。

> ipam
>- host-local：基于本地文件的 ip 分配和管理，把分配的 IP 地址保存在文件中
>- dhcp：从已经运行的 DHCP 服务器中获取 ip 地址

### kubelet如何调用CNI

[kubelet调用CNI结构图](https://ask.qcloudimg.com/http-save/1319879/v7z4zcvwc1.png)
[kubelet调用CNI模型](http://upload-images.jianshu.io/upload_images/3611024-8026258c0424f952.png)

在容器运行时(CRI)，会调用CNI插件为sanbox容器配置网络

[kubelet调用CNI流程图](http://upload-images.jianshu.io/upload_images/3611024-c344ae56935dc1b5.png)

### 参考文档
- https://cloud.tencent.com/developer/article/1579809?s=original-sharing   //Extend Kubernetes - CNI
- https://wiki.opskumu.com/kubernetes/wang-luo-fang-an/src-kubelet-cni //Kubelet CNI 源码解析
- https://zhuanlan.zhihu.com/p/34085344  //kubernetes源码阅读 kubelet对cni的实现
- https://www.jianshu.com/p/1919fb8a48ea  //Kubelet 对CNI的实现
- https://yucs.github.io/2017/12/06/2017-12-6-CNI/  //Kubernetes网络插件CNI调研整理
- https://blog.csdn.net/waltonwang/article/details/72669826  //从源码看kubernetes与CNI Plugin的集成
