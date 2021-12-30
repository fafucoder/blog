---
title: kubernetes dns解析
date: 2020-07-08 16:44:29
tags:
- kubernetes
categories:
- kubernetes
---

### DNS概念

1. DNS协议的作用是将域名解析成ip地址， 域名只是ip地址的代号

2. 为了将域名解析成ip地址，需要建立一个域名到ip地址之间的映射关系，提供将域名动态解析成ip地址的服务器就叫做DNS服务器， 由于当前故障的原因，DNS无法设计成集中式，因此出现了分层解析架构，分为根服务器，二级域名服务器，三级域名服务器等。
- 根域名服务器（Root Domain）：用 · （句点）表示，根域名服务器全球的数量是固定的，为 13 个（当然这里的“个”是“组”的意思）

- 顶级域名服务器（Top-Level Domain, TLD）：指代某个国家/地区或组织使用的类型名称，如 com、cn、edu 等。

- 次级域名服务器（Second-Level Domain, SLD）：个人或组织在 Internet 上注册的名称，如 qq.com、gitHub.com 等

- 三次域名服务器或权威域名服务器（如果有）：这层严格来说是次级域名的子域，是二层域名派生的域名，通俗说就是网站名，如 index.baidu.com
3. linux网络中，`/etc/hosts`用于记录主机hostname跟ip之间的映射关系，/etc/resolve.conf用于记录外网DNS服务器地址

### DNS查询过程

![DNS查询](https://tva1.sinaimg.cn/large/007S8ZIlgy1ghpj97njjzj30ze0ki12v.jpg)

以查询`www.baidu.com`为例子:

1. 本地终端发出域名解析请求，到本地DNS服务器
2. 本地DNS服务器查询DNS缓存是否有对应的Ip, 检查hosts文件中是否与对应的ip地址, 不存在就想外部DNS服务器发送请求(Dnsmasq提供DNS缓存和DHCP服务、Tftp服务功能。当接受到一个DNS请求时，Dnsmasq首先会查找/etc/hosts这个文件，然后查找/etc/resolv.conf中定义的外部DNS)
3. 外部DNS服务器同样先查看缓存，不存在再向众所周知的全球 13 台根服务器发出请求
4. 根 DNS 服务器收到请求后会判断该域名由谁授权管理，返回管理这个域名的顶级 DNS 服务器的 IP，给到DNS服务器
5. DNS 服务器根据返回的 IP 继续请求顶级 DNS 服务器
6. 顶级 DNS 服务器收到请求后，如果自己无法解析，也会判断对应域名由下一级的哪个 DNS 服务器授权管理，并将该次级 DNS 服务器的 IP 返回
7. DNS 服务器继续根据返回的 IP 请求次级 DNS 服务器
8. 次级 DNS 服务器如果解析出对应的域名，就将该域名对应的 IP 返回给 DNS 服务器
9. DNS 服务器缓存一份域名与 IP 的映射关系

### DNS 域名记录设置

在使用云厂商的域名后进行DNS域名配置时，首先需要选择对应的记录类型，DNS有多种记录类型，包括：

- **A 地址记录， 将域名指向一个 IP 地址(例如：120.55.23.197)；**
- **AAAA 地址记录， 将域名指向一个 IPv6 地址；**
- **CNAME 别名记录，将域名指向另一个域名，再由另一个域名提供 IP 地址；**
- **TXT 域名对应的文本信息， 对域名进行标识和说明，绝大多数的 TXT 记录是用来做 SPF 记录(反垃圾邮件);**
- **NS 名字服务器记录, 将子域名交给其他 DNS 服务商解析；**
- **SRV TCP服务器信息记录， 用来标识某台服务器使用了某个服务，常见于微软系统的目录管理;**
- **MX 邮件服务器记录， 设置邮箱，让邮箱能收到邮件;**
- AFSDB Andrew文件系统数据库服务器记录；
- ATMA ATM地址记录；
- HINFO 硬件配置记录，包括CPU、操作系统信息;
- ISDN 域名对应的ISDN号码;
- MB 存放指定邮箱的服务器;
- MG 邮件组记录;
- MINFO 邮件组和邮箱的信息记录;
- MR 改名的邮箱记录;
- PTR 反向记录;
- RP 负责人记录;
- RT 路由穿透记录;
- X25 域名对应的X.25地址记录;

![DNS记录类型](https://tva1.sinaimg.cn/large/008i3skNly1gxr2jopyinj31l60u0gog.jpg)

> 注： 主机记录就是域名前缀，例如`www.baidu.com`中的www就是域名前缀，比较特殊的主机记录有：
> 
> 1. **@**：直接解析主域名，例如@.baidu.com == https://baidu.com
> 2. ***** :   泛解析，匹配其他所有域名, 例如 *.baidu.com == https://xxxx.baidu.com

### DNS 查询相关的命令

#### 1、nslookup 命令

nslookup 是域名查询命令， 用户查询域名对应的ip地址， 命令格式如下：

`nslookup damain [dns-server]`

如果未指定dns服务器时， 采用系统默认的dns服务器

```
dawn@node-2:~$ nslookup baidu.com
Server:        127.0.0.53
Address:    127.0.0.53#53

Non-authoritative answer:  #未认证回答
Name:    baidu.com
Address: 39.156.69.79
Name:    baidu.com
Address: 220.181.38.148
```

nslookup 默认查询的A类型的记录， 可以设置查询指定类型的记录，通过`qt=类型`查询指定类型的记录，例如：`nslookup -qt=mx baidu.com`

#### 2、whois 命令

whois用于查询域名的注册情况。

```
[root@whois~]# whois baidu.com
   Domain Name: BAIDU.COM
   Registry Domain ID: 11181110_DOMAIN_COM-VRSN
   Registrar WHOIS Server: whois.markmonitor.com
   Registrar URL: http://www.markmonitor.com
   Updated Date: 2019-05-09T04:30:46Z
   Creation Date: 1999-10-11T11:05:17Z
   Registry Expiry Date: 2026-10-11T11:05:17Z
   Registrar: MarkMonitor Inc.
   Registrar IANA ID: 292
   Registrar Abuse Contact Email: abusecomplaints@markmonitor.com
   Registrar Abuse Contact Phone: +1.2083895740
   Domain Status: clientDeleteProhibited https://icann.org/epp#clientDeleteProhibited
   Domain Status: clientTransferProhibited https://icann.org/epp#clientTransferProhibited
   Domain Status: clientUpdateProhibited https://icann.org/epp#clientUpdateProhibited
   Domain Status: serverDeleteProhibited https://icann.org/epp#serverDeleteProhibited
   Domain Status: serverTransferProhibited https://icann.org/epp#serverTransferProhibited
   Domain Status: serverUpdateProhibited https://icann.org/epp#serverUpdateProhibited
   Name Server: NS1.BAIDU.COM
   Name Server: NS2.BAIDU.COM
   Name Server: NS3.BAIDU.COM
   Name Server: NS4.BAIDU.COM
   Name Server: NS7.BAIDU.COM
   DNSSEC: unsigned
   URL of the ICANN Whois Inaccuracy Complaint Form: https://www.icann.org/wicf/
```

### 3、 dig 命令

Linux下解析域名除了使用nslookup之外，还可以使用dig(全称 `domain information groper`)命令来解析域名，相对于nslookup, dig命令的输出更为丰富，它会打印出>DNS name server的回应。dig命令的格式如下：

```linux
dig [@server] [-b address] [-c class] [-f filename] [-k filename] [ -n ][-p port#] [-t type] [-x addr] [-y name:key] [name] [type] [class] [queryopt...]
dig [global-queryopt...] [query...]
```

选项：

```
@server：指定进行域名解析的域名服务器；
-b [address]：当主机具有多个IP地址，指定使用本机的哪个IP地址向域名服务器发送域名查询请求；
-f [filename]：指定dig以批处理的方式运行，指定的文件中保存着需要批处理查询的DNS任务信息；
-P [port]：指定域名服务器所使用端口号；
-t [type]：指定要查询的DNS数据类型，缺省查询类型为A记录；
-x [address]：执行逆向域名查询(将地址映射到名称)；
-4：使用IPv4；
-6：使用IPv6；
```

查询选项：

```linux
+[no]tcp: 查询域名服务器时使用 [不使用] TCP。缺省行为是使用 UDP，除非是 AXFR 或 IXFR 请求，才使用 TCP 连接。 
+[no]vc:  查询名称服务器时使用 [不使用] TCP。+[no]tcp 的备用语法提供了向下兼容。vc 代表虚电路。 
+[no]ignore: 忽略 UDP 响应的中断，而不是用 TCP 重试。缺省情况运行 TCP 重试。 
+domain=somename: 设定包含单个域 somename 的搜索列表，好像被 /etc/resolv.conf 中的域伪指令指定，并且启用搜索列表处理，好像给定了 +search 选项。 
+[no]adflag: 在查询中设置 [不设置] AD（真实数据）位。目前 AD 位只在响应中有标准含义，而查询中没有，但是出于完整性考虑在查询中这种性能可以设置。 
+[no]cdflag: 在查询中设置 [不设置] CD（检查禁用）位。它请求服务器不运行响应信息的 DNSSEC 合法性。 
+[no]recursive: 切换查询中的 RD（要求递归）位设置。在缺省情况下设置该位，也就是说 dig 正常情形下发送递归查询。当使用查询选项 +nssearch 或 +trace 时，递归自动禁用。 
+[no]nssearch: 这个选项被设置时，dig 试图寻找包含待搜名称的网段的权威域名服务器，并显示网段中每台域名服务器的 SOA 记录。 
+[no]trace: 切换为待查询名称从根名称服务器开始的代理路径跟踪。缺省情况不使用跟踪。一旦启用跟踪，dig 使用迭代查询解析待查询名称。它将按照从根服务器的参照，显示来自每台使用解析查询的服务器的应答。 
+[no]cmd: 设定在输出中显示指出 dig 版本及其所用的查询选项的初始注释。缺省情况下显示注释。 
+[no]short: 提供简要答复。缺省值是以冗长格式显示答复信息。  
+[no]comments: 切换输出中的注释行显示。缺省值是显示注释。 
+[no]stats: 查询选项设定显示统计信息：查询进行时，应答的大小等等。缺省显示查询统计信息。 
+[no]qr: 显示 [不显示] 发送的查询请求。缺省不显示。 
+[no]question: 当返回应答时，显示 [不显示] 查询请求的问题部分。缺省作为注释显示问题部分。 
+[no]answer: 显示 [不显示] 应答的回答部分。缺省显示。 
+[no]authority: 显示 [不显示] 应答的权限部分。缺省显示。 
+[no]additional: 显示 [不显示] 应答的附加部分。缺省显示。 
+[no]all: 设置或清除所有显示标志。 
+time=T: 为查询设置超时时间为T秒。缺省是5秒。如果将T设置为小于1的数，则以1秒作为查询超时时间。 
+tries=A: 设置向服务器发送UDP查询请求的重试次数为A，代替缺省的3次。如果把A小于或等于0，则采用1为重试次数。 
+ndots=D: 出于完全考虑，设置必须出现在名称 D 的点数。缺省值是使用在 /etc/resolv.conf 中的 ndots 语句定义的，或者是 1，如果没有 ndots 语句的话。带更少点数的名称被解释为相对名称，并通过搜索列表中的域或文件 /etc/resolv.conf 中的域伪指令进行搜索。 
+bufsize=B: 设置使用 EDNS0 的 UDP 消息缓冲区大小为 B 字节。缓冲区的最大值和最小值分别为 65535 和 0。超出这个范围的值自动舍入到最近的有效值。 
+[no]multiline: 以详细的多行格式显示类似 SOA 的记录，并附带可读注释。缺省值是每单个行上显示一条记录，以便于计算机解析 dig 的输出。 
```

查询例子：

```
[root@localhost ~]# dig www.baidu.com

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.8 <<>> www.baidu.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 35946
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 5, ADDITIONAL: 5

;; QUESTION SECTION:
;www.baidu.com.                 IN      A

;; ANSWER SECTION:
www.baidu.com.          837     IN      CNAME   www.a.shifen.com.
www.a.shifen.com.       600     IN      A       14.215.177.39
www.a.shifen.com.       600     IN      A       14.215.177.38

;; AUTHORITY SECTION:
a.shifen.com.           151     IN      NS      ns4.a.shifen.com.
a.shifen.com.           151     IN      NS      ns3.a.shifen.com.
a.shifen.com.           151     IN      NS      ns2.a.shifen.com.
a.shifen.com.           151     IN      NS      ns5.a.shifen.com.
a.shifen.com.           151     IN      NS      ns1.a.shifen.com.

;; ADDITIONAL SECTION:
ns1.a.shifen.com.       143     IN      A       110.242.68.42
ns2.a.shifen.com.       305     IN      A       220.181.33.32
ns3.a.shifen.com.       305     IN      A       112.80.255.253
ns4.a.shifen.com.       273     IN      A       14.215.177.229
ns5.a.shifen.com.       443     IN      A       180.76.76.95

;; Query time: 4 msec
;; SERVER: 218.85.152.99#53(218.85.152.99)
;; WHEN: Sat Dec 25 00:37:02 EST 2021
;; MSG SIZE  rcvd: 260
```

### DNS In Kubernetes

在 k8s 中，一个 Pod 如果要访问同 Namespace 下的 Service（比如 user-svc），那么只需要curl user-svc。如果 Pod 和 Service 不在同一域名下，那么就需要在 Service Name 之后添加上 Service 所在的 Namespace（比如 beta），curl user-svc.beta。那么 k8s 是如何知道这些域名是内部域名并为他们做解析的呢？答案是dns

#### resolv.conf

resolv.conf 是 DNS 域名解析的配置文件。每行都会以一个关键字开头，然后跟配置参数。这里主要使用到的关键词有3个。

- nameserver #定义 DNS 服务器的 IP 地址
- search #定义域名的搜索列表，当查询的域名中包含的 . 的数量少于 options.ndots 的值时，会依次匹配列表中的每个值
- options #定义域名查找时的配置信息

那么我们进入一个 Pod 查看它的 resolv.conf

```
1. nameserver 100.64.0.10
2. search default.svc.cluster.local svc.cluster.local cluster.local
3. options ndots:5
```

上述配置文件 resolv.conf 是 dnsPolicy: ClusterFirst 情况下，k8s 为 Pod 自动生成的，这里的 nameserver 所对应的地址正是 DNS Service 的Cluster IP（该值在启动 kubelet 的时候，通过 clusterDNS 指定）。所以，从集群内请求的所有的域名解析都需要经过 DNS Service 进行解析，不管是 k8s 内部域名还是外部域名。

可以看到这里的 search 域默认包含了 namespace.svc.cluster.local、svc.cluster.local 和 cluster.local 三种。当我们在 Pod 中访问 a Service时（ curl a ），会选择nameserver 100.64.0.10 进行解析，然后依次带入 search 域进行 DNS 查找，直到找到为止。

```
# curl a
a.default.svc.cluster.local
```

显然因为 Pod 和 a Service 在同一 Namespace 下，所以第一次 lookup 就能找到。

如果 Pod 要访问不同 Namespace（例如： beta ）下的 Service b （ curl b.beta ），会经过两次 DNS 查找，分别是

```
# curl b.beta
1. b.beta.default.svc.cluster.local（未找到）
2. b.beta.svc.cluster.local（找到）
```

正是因为 search 的顺序性，所以访问同一 Namespace 下的 Service， curl a 是要比 curl a.default 的效率更高的，因为后者多经过了一次 DNS 解析。

#### 那么当Pod中访问外部域名时仍然需要走search域吗

这个答案，不能说肯定也不能说否定，看情况，可以说，大部分情况要走 search 域。

在 /etc/resolv.conf 文件中，我们可以看到 options 中有个配置项 ndots:5 。

ndots:5，表示：如果需要 lookup 的 Domain 中包含少于5个 . ，那么将使用非绝对域名，如果需要查询的 DNS 中包含大于或等于5个 . ，那么就会使用绝对域名。如果是绝对域名则不会走 search 域，如果是非绝对域名，就会按照 search 域中进行逐一匹配查询，如果 search 走完了都没有找到，那么就会使用 原域名.（domgoer.com.） 的方式作为绝对域名进行查找。
在 /etc/resolv.conf 文件中，我们可以看到 options 中有个配置项 ndots:5 。

#### 因此,可以找到两种优化的方法

1. 直接使用绝对域名

这是最简单直接的优化方式，可以直接在要访问的域名后面加上 . 如：domgoer.com. ，这样就会避免走 search 域进行匹配。

2. 配置ndots

还记得之前说过 /etc/resolv.conf 中的参数都可以通过k8s中的 dnsConfig 字段进行配置。这就允许你根据你自己的需求配置域名解析的规则。

例如 当域名中包含两个 . 或以上时，就能使用绝对域名直接进行域名解析。

```
apiVersion: v1
kind: Pod
metadata:
namespace: default
name: dns-example
spec:
containers:
- name: test
  image: nginx
dnsConfig:
  options:
  - name: ndots
    value: 2
```

### Kubernetes DNS配置

Pod中dns配置主要有dnsConfig跟dnsPolicy, 其中dnsPolicy主要dns网络策略， dnsConfig主要配置dns的resolv.conf配置，详情可以参考[官方api](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#podspec-v1-core)

![DNS配置](https://tva1.sinaimg.cn/large/007S8ZIlgy1ghpl88nr9sj323608yn0i.jpg)

#### dnsConfig

dnsConfig通过配置nameserver、search 和 options来修改容器的pod的resolv.conf配置

```
dnsConfig:
    nameservers:
      - 1.2.3.4
    searches:
      - ns1.svc.cluster.local
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "2"
      - name: edns0
```

#### dnsPolicy

在k8s中，有4中DNS策略，分别是 `ClusterFirstWithHostNet`、`ClusterFirst`、`Default`、和 `None`，这些策略可以通过dnsPolicy这个字段来定义

如果在初始化 Pod、Deployment 或者 RC 等资源时没有定义，则会默认使用 ClusterFirst 策略

1. ClusterFirstWithHostNet

        当一个 Pod 以 HOST 模式（和宿主机共享网络）启动时，这个 POD 中的所有容器都会使用宿主机的/etc/resolv.conf 配置进行 DNS 查询，但是如果你还想继续使用 Kubernetes 的 DNS 服务，
就需要将 dnsPolicy 设置为 ClusterFirstWithHostNet。

2. ClusterFirst

        使用这是方式表示 Pod 内的 DNS 优先会使用 k8s 集群内的DNS服务，也就是会使用 kubedns 或者 coredns 进行域名解析。如果解析不成功，才会使用宿主机的 DNS 配置进行解析。

3. Default

        这种方式，会让 kubelet 来绝定 Pod 内的 DNS 使用哪种 DNS 策略。kubelet 的默认方式，其实就是使用宿主机的 /etc/resolv.conf 来进行解析。你可以通过设置 kubelet 的启动参数，–resolv-conf=/etc/resolv.conf 来决定 DNS 解析文件的地址

4. None

        这种方式顾名思义，不会使用集群和宿主机的 DNS 策略。而是和 dnsConfig 配合一起使用，来自定义 DNS 配置，否则在提交修改时报错。

### 参考文档

- https://blog.domgoer.io/2019/08/07/kube-dns-and-core-dns
- https://cizixs.com/2017/04/11/kubernetes-intro-kube-dns/
- https://juejin.im/entry/5b84a90f51882542e60663cc
- https://kubernetes.io/zh/docs/concepts/services-networking/dns-pod-service/
- https://mp.weixin.qq.com/s/V5qOl5Ure3Pj5aZmTFp-Fg
- https://sanyuesha.com/2017/11/08/how-dns-resolve-in-linux/