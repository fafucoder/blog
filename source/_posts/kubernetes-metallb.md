---
title: kubernetes metallb 原理解析
date: 2020-08-06 11:02:25
tags:
- kubernetes
categories:
- kubernetes
---

### 概述
k8s的LoadBalancer类型的Service依赖于外部的云提供的Load Balancer。为了从外部访问裸机 Kubernetes 群集，目前只能使用 NodePort 或 Ingress 的方法进行服务暴露。前者的缺点是每个暴露的服务需要占用所有节点的某个端口，后者的缺点是仅仅能支持 HTTP 协议。

[MetalLB](https://metallb.universe.tf/) 是一个负载均衡器，专门解决裸金属 Kubernetes 集群中无法使用 LoadBalancer 类型服务的痛点。

### 工作原理
MetalLB 会在 Kubernetes 内运行，监控服务对象的变化，一旦监测到有新的 LoadBalancer 服务运行，并且没有可申请的负载均衡器之后，就会完成地址分配和外部声明两部分的工作。使用 MetalLB 时，MetalLB 会自己为用户的 LoadBalancer 类型 Service 分配 IP 地址，当然该 IP 地址不是凭空产生的，需要用户在配置中提供一个 IP 地址池，Metallb 将会在其中选取地址分配给服务。

MetalLB 将 IP 分配给某个服务后，它需要对外宣告此 IP 地址，并让外部主机可以路由到此 IP。MetalLB 支持两种声明模式：Layer 2（ ARP / NDP ）模式或者 BGP 模式。

#### Layer2 模式
![Layer2模式](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/007S8ZIlgy1gjmutv3mxgj312i0ng46m.jpg)

Layer2 模式下，每个 Service 会有集群中的一个 Node 来负责。服务的入口流量全部经由单个节点，然后该节点的 Kube-Proxy 会把流量再转发给服务的 Pods。也就是说，该模式下 MetalLB 并没有真正提供负载均衡器。尽管如此，MetalLB 提供了故障转移功能，如果持有 IP 的节点出现故障，则默认 10 秒后即发生故障转移，IP 会被分配给其它健康的节点。

Layer2 模式的优缺点：
1. Layer2 模式更为通用，不需要用户有额外的设备；
2. Layer2 模式下存在单点问题，服务的所有入口流量经由单点，其网络带宽可能成为瓶颈；
3. 由于 Layer 2 模式需要 ARP/NDP 客户端配合，当故障转移发生时，MetalLB 会发送 ARP 包来宣告 MAC 地址和 IP 映射关系的变化，地址分配略为繁琐。

#### BGP模式
![BGP模式](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/007S8ZIlly1gjmuxncponj314q0nsk0k.jpg)

当在第三层工作时，集群中所有机器都和你控制的最接近的路由器建立 BGP 会话，此会话让路由器能学习到如何转发针对 K8S 服务 IP 的数据包。

通过使用 BGP，可以实现真正的跨多节点负载均衡（需要路由器支持 multipath），还可以基于 BGP 的策略机制实现细粒度的流量控制。

具体的负载均衡行为和路由器有关，可保证的共同行为是：每个连接（TCP 或 UDP 会话）的数据包总是路由到同一个节点上。

BGP 模式的优缺点：
1. 不能优雅处理故障转移，当持有服务的节点宕掉后，所有活动连接的客户端将收到 Connection reset by peer ；
2. BGP 路由器对数据包的源 IP、目的 IP、协议类型进行简单的哈希，并依据哈希值决定发给哪个 K8S 节点。问题是 K8S 节点集是不稳定的，一旦（参与 BGP）的节点宕掉，很大部分的活动连接都会因为 rehash 而坏掉。

BGP 模式问题的缓和措施：
1. 将服务绑定到一部分固定的节点上，降低 rehash 的概率。
2. 在流量低的时段改变服务的部署。
3. 客户端添加透明重试逻辑，当发现连接 TCP 层错误时自动重试。

### 运行流程
![运行流程](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/007S8ZIlgy1gjmt9fchy8j31fz0u0dq7.jpg)

在metallb中，controller跟speaker通过list-watch service, configmap跟api-service交互，当创建一个loadbalancer的service的时候，controller获取到svc，检查configMap是否有config配置，接着为svc分配一个地址，最后更新svc的status(更新svc.Status.LoadBalancer.Ingress), 此时svc已经获取到了一个external ip地址。

speaker是一个daemonset，在speaker中，所以speaker会初始化[memberlist](github.com/hashicorp/memberlist), 然后把所有speaker所在node加入到memberlist中，目的是实现节点快速的故障转移(如果使用node list-watch机制，sync其实有个时间过程，而使用memberlist能更快速的故障转移)。 当controller更新完状态后，speaker监听到资源变化，获取到loadbalancerIP，  选择合适的节点进行通告(选择backends所在node作为通告节点，首先会获取到endpoints->接着endpoints的NodeName->判断memberlist中是否有此nodeName->nodeName作为通告节点), 如果loadbalanceIP是ipv4则选择arp进行通告，否则就选择ndp进行通告。
```
func (a *Announce) SetBalancer(name string, ip net.IP) {
	a.Lock()
	defer a.Unlock()

	// Kubernetes may inform us that we should advertise this address multiple
	// times, so just no-op any subsequent requests.
	if _, ok := a.ips[name]; ok {
		return
	}
	a.ips[name] = ip

	a.ipRefcnt[ip.String()]++
	if a.ipRefcnt[ip.String()] > 1 {
		// Multiple services are using this IP, so there's nothing
		// else to do right now.
		return
	}

	for _, client := range a.ndps {
		if err := client.Watch(ip); err != nil {
			a.logger.Log("op", "watchMulticastGroup", "error", err, "ip", ip, "msg", "failed to watch NDP multicast group for IP, NDP responder will not respond to requests for this address")
		}
	}

	go a.spam(name)
}

func (a *Announce) spam(name string) {
	start := time.Now()
	for time.Since(start) < 5*time.Second {
		if err := a.gratuitous(name); err != nil {
			a.logger.Log("op", "gratuitousAnnounce", "error", err, "service", name, "msg", "failed to make gratuitous IP announcement")
		}
		time.Sleep(1100 * time.Millisecond)
	}
}

func (a *Announce) gratuitous(name string) error {
	a.Lock()
	defer a.Unlock()

	ip, ok := a.ips[name]
	if !ok {
		// No IP means we've lost control of the IP, someone else is
		// doing announcements.
		return nil
	}
	if ip.To4() != nil {
		for _, client := range a.arps {
			if err := client.Gratuitous(ip); err != nil {
				return err
			}
		}
	} else {
		for _, client := range a.ndps {
			if err := client.Gratuitous(ip); err != nil {
				return err
			}
		}
	}
	return nil
}

func (a *arpResponder) Gratuitous(ip net.IP) error {
	for _, op := range []arp.Operation{arp.OperationRequest, arp.OperationReply} {
		pkt, err := arp.NewPacket(op, a.hardwareAddr, ip, ethernet.Broadcast, ip)
		if err != nil {
			return fmt.Errorf("assembling %q gratuitous packet for %q: %s", op, ip, err)
		}
		if err = a.conn.WriteTo(pkt, ethernet.Broadcast); err != nil {
			return fmt.Errorf("writing %q gratuitous packet for %q: %s", op, ip, err)
		}
		stats.SentGratuitous(ip.String())
	}
	return nil
}
```
speaker通告过程如下，首先在speaker初始化中，会获取到主机的所有网卡，然后过滤掉没有地址，状态不是UP，ARP未开启的网卡，然后获取这些网卡的第一个IP地址(一个网卡可能有多个地址), 使用rawSocket监听网卡的数据包，构建应答包。
```
func newARPResponder(logger log.Logger, ifi *net.Interface, ann announceFunc) (*arpResponder, error) {
	client, err := arp.Dial(ifi)
	if err != nil {
		return nil, fmt.Errorf("creating ARP responder for %q: %s", ifi.Name, err)
	}

	ret := &arpResponder{
		logger:       logger,
		intf:         ifi.Name,
		hardwareAddr: ifi.HardwareAddr,
		conn:         client,
		closed:       make(chan struct{}),
		announce:     ann,
	}
	go ret.run()
	return ret, nil
}

func (a *arpResponder) run() {
	for a.processRequest() != dropReasonClosed {
	}
}

func (a *arpResponder) processRequest() dropReason {
	pkt, eth, err := a.conn.Read()
	if err != nil {
		// ARP listener doesn't cleanly return EOF when closed, so we
		// need to hook into the call to arpResponder.Close()
		// independently.
		select {
		case <-a.closed:
			return dropReasonClosed
		default:
		}
		if err == io.EOF {
			return dropReasonClosed
		}
		return dropReasonError
	}

	// Ignore ARP replies.
	if pkt.Operation != arp.OperationRequest {
		return dropReasonARPReply
	}

	// Ignore ARP requests which are not broadcast or bound directly for this machine.
	if !bytes.Equal(eth.Destination, ethernet.Broadcast) && !bytes.Equal(eth.Destination, a.hardwareAddr) {
		return dropReasonEthernetDestination
	}

	// Ignore ARP requests that the announcer tells us to ignore.
	if reason := a.announce(pkt.TargetIP); reason != dropReasonNone {
		return reason
	}

	stats.GotRequest(pkt.TargetIP.String())
	a.logger.Log("interface", a.intf, "ip", pkt.TargetIP, "senderIP", pkt.SenderIP, "senderMAC", pkt.SenderHardwareAddr, "responseMAC", a.hardwareAddr, "msg", "got ARP request for service IP, sending response")

	if err := a.conn.Reply(pkt, a.hardwareAddr, pkt.TargetIP); err != nil {
		a.logger.Log("op", "arpReply", "interface", a.intf, "ip", pkt.TargetIP, "senderIP", pkt.SenderIP, "senderMAC", pkt.SenderHardwareAddr, "responseMAC", a.hardwareAddr, "error", err, "msg", "failed to send ARP reply")
	} else {
		stats.SentResponse(pkt.TargetIP.String())
	}
	return dropReasonNone
}
```

### 参考文档
- https://zhuanlan.zhihu.com/p/103717169
- https://blog.fleeto.us/post/intro-metallb/
- https://github.com/metallb/metallb
- https://cloud.tencent.com/developer/article/1631910
- https://www.jianshu.com/p/5e39d1637184