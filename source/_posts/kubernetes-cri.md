---
title: kubernetes容器运行时containerd
date: 2020-01-31 15:46:25
tags:
- kubernetes
- cri
- containerd
---

### k8s工作原理

在 K8s 中，存在一个控制面板，也就是我们所说的 master node， 上面运行着 apiserver、controllerManager、kubeScheduler、kubedns 等组件。当我们想要创建一个应用（deployment、statefulset）时，主要流程如下：

![k8s工作原理](https://user-gold-cdn.xitu.io/2020/1/15/16fa82e999e6e874?imageslim.jpg)

具体的流程如下：
1. 通过 kubectl 命令向 apiserver 提交， apiserver 将资源保存在 etcd 中
2. controller manager 通过控制循环，获取新创建的资源，并创建 pod 信息。注意这里只创建pod，并未调度和创建容器
3. kube-scheduler 也会循环获取新创建但未调度的pod，并在执行一系列调度算法后，将 pod 绑定到一个 node上，并更新 etcd 中的信息。具体方式是在 pod 的 spec 中加入 nodeName 字段。
4. Kubelet监视所有Pod对象的更改。当发现Pod已绑定到Node，并且绑定的Node本身时，Kubelet会接管所有后续任务，包括创建 pod 网络，container等。
5. kubelet通过 CRI 调用 container runtime 创建 pod 中的 container。

引入cri后 kubelet架构图：
![kubelet架构图](https://feisky.gitbooks.io/kubernetes/plugins/assets/image-20190316183052101.png)

kubelet调用具体的cri流程如下(以docker为例)：

1. kubelet 通过 CRI(Container Runtime Interface) 接口(gRPC) 调用 docker shim, 请求创建一个容器, 这一步中, Kubelet 可以视作一个简单的 CRI Client, 而 docker shim 就是接收请求的 Server

2. docker shim 收到请求后, 通过适配的方式，适配成 Docker Daemon 的请求格式, 发到 Docker Daemon 上请求创建一个容器。在docker 1.12后版本中，docker daemon被拆分成dockerd和containerd，containerd负责操作容器
3. dockerd收到请求后， 调用containerd进程去创建一个容器

4. containerd 收到请求后, 并不会自己直接去操作容器, 而是创建一个叫做 containerd-shim 的进程, 让 containerd-shim 去操作容器

5. containerd-shim 在这一步需要调用 runC 这个命令行工具, 来启动容器，runC是OCI(Open Container Initiative, 开放容器标准) 的一个参考实现。主要用来设置 namespaces 和 cgroups, 挂载 root filesystem等操作
6. runC启动完容器后本身会直接退出, containerd-shim 则会成为容器进程的父进程, 负责收集容器进程的状态, 上报给 containerd, 并在容器中 pid 为 1 的进程退出后接管容器中的子进程进行清理, 确保不会出现僵尸进程（关闭进程描述符等）
![cri运行流程](https://img2018.cnblogs.com/blog/595328/201909/595328-20190924162334102-1814702716.png)

### 为何要引入CRI

为了让Kubernetes不和某种特定的容器运行时技术绑死，而是能无需重新编译源代码就能够支持多种容器运行时技术的替换，和我们面向对象设计中引入接口作为抽象层一样，在Kubernetes和容器运行时之间我们引入了一个抽象层，即容器运行时接口。

![k8s工作原理](https://img-blog.csdnimg.cn/2018121911421288.png)

在CRI还没有问世的Kubernetes早期版本里，比如1.3版本里，添加了对另一个容器运行时技术rkt的支持，即rktnetes项目。

这个项目虽然让Kubernetes增加了除Docker之外的另一种容器运行时的支持，然而这种增强的实现方式是通过直接修改kubelet实现源代码进行的，需要贡献者非常熟悉kubelet内部原理，开发门槛较高。

为了实现一个真正支持可插拔替换的容器运行时的机制，Kubernetes引入了CRI的概念。

有了CRI后，kubelet不再直接和容器运行时交互，而是通过CRI这个中间层。

kubelet和CRI通过Unix 套接字或者gRPC框架进行通信。

### OCI概念

Open Container Initiative，也就是常说的OCI，是由多家公司共同成立的项目，并由linux基金会进行管理，致力于container runtime的标准的制定和runc的开发等工作。

所谓container runtime，主要负责的是容器的生命周期的管理。oci的runtime spec标准中对于容器的状态描述，以及对于容器的创建、删除、查看等操作进行了定义。

runc，是对于OCI标准的一个参考实现，是一个可以用于创建和运行容器的CLI(command-line interface)工具。runc直接与容器所依赖的cgroup/linux kernel等进行交互，负责为容器配置cgroup/namespace等启动容器所需的环境，创建启动容器的相关进程。

为了兼容oci标准，docker也做了架构调整。将容器运行时相关的程序从docker daemon剥离出来，形成了containerd。Containerd向docker提供运行容器的API，二者通过grpc进行交互。containerd最后会通过runc来实际运行容器。

![docker oci](http://xuxinkun.github.io/img/docker-oci-runc-k8s/containerd.png)

为了进一步与oci进行兼容，kubernetes还孵化了cri-o，成为了架设在cri和oci之间的一座桥梁。通过这种方式，可以方便更多符合oci标准的容器运行时，接入kubernetes进行集成使用。

![k8s oci](http://xuxinkun.github.io/img/docker-oci-runc-k8s/kubelet.png)

上面的图片代表容器运行时的四个阶段(从右往左):
1. 最早是在kubelet这一层进行适配，通过启动docker-manager来访问docker（不同的容器产品有不同的manager，这部分代码是kubelet项目标准代码的一部分）

2. K8 1.5之后引入了CRI接口，通过docker-shim（垫片）的方式接入docker。shim程序一般由容器厂商根据CRI规范自己开发，实现方式可以自己定义（即CRI规范定义了要做什么，怎么做可以基于自己的理解）。而docker-shim由于历史原因，还是k8项目组来做的，这部分代码也包含在kubelet代码里面，但架构上是分开的。

3. docker在分出containerd后，k8也顺应潮流，孵化了cri-containerd项目，用于和containerd对接，这样就不走docker daemon了。

4. 目前孵化的cri-o项目，则连containerd都绕过了，直接使用runC去创建容器（你可以把cri-o看作是k8生态里面的containerd）

### containerd

在Containerd1.0 及以前版本将 dockershim 和 docker daemon 替换为 cri-containerd + containerd，而在 1.1 版本直接将 cri-containerd 内置在 Containerd 中，简化为一个 CRI 插件。
![k8s containerd](https://feisky.gitbooks.io/kubernetes/plugins/images/cri-containerd.png)

Containerd 内置的 CRI 插件实现了 Kubelet CRI 接口中的 Image Service 和 Runtime Service，通过内部接口管理容器和镜像，并通过 CNI 插件给 Pod 配置网络。
![kubelet containerd](https://feisky.gitbooks.io/kubernetes/plugins/images/containerd.png)

### CRI运行原理

![cri接口规范](https://img-blog.csdnimg.cn/20190407201430191.jpg)

CRI 基于 gRPC 定义了 RuntimeService 和 ImageService 等两个 gRPC 服务，分别用于容器运行时和镜像的管理

RuntimeService: 它提供的接口,主要就是和容器有关的操作.比如,创建和启动容器,删除容器,执行 exec 命令等

ImageService: 它提供的接口,主要是和容器镜像相关的操作,比如 拉取镜像,删除镜像等等

### CRI工作流程

![cri工作流程](https://img-blog.csdnimg.cn/20190406213718422.jpg)

#### 创建容器

1. kubelet会通过 grpc 调用 CRI 接口，首先去创建一个环境，也就是所谓的 PodSandbox（pause容器）
2. 当 PodSandbox 可用后，继续调用 Image 或 Container 接口去拉取镜像和创建容器

#### 运行容器

CRI 机制能够发挥作用的核心,在于每一个容器项目现在都可以自己实现一个 CRI shim ,自行对 CRI 请求进行处理.这样, Kubernetes 就有了一个统一的容器抽象层,使得下层容器在运行的时候,可以自由地对接,从而进入 Kubernetes 当中去.

CRI shim 还有一个重要的工作,就是如何实现 exec , logs 等接口.这些接口不同在于, gRPC 接口调用期间, kubelet 需要和容器项目维护一个长连接来传输数据.这种 API ,就称之为 Streaming API .

CRI shim 中对 Streaming API 的实现,依赖于一套独立的 Streaming Server 机制如下： 
![cri工作流程](https://feisky.gitbooks.io/kubernetes/plugins/assets/image-20190316183005314.png)

1. 当对一个容器执行 kubectl exec 命令时,这个请求首先会交给 API Server.

2. API Server 就会调用 kubelet 的 Exec API .此时, kubelet 会调用 CRI 的 Exec 接口,而负责响应这个接口的,就是 CRI shim

3. CRI shim 并不会直接去调用后端的容器项目(比如 Docker )来进行处理,而只会返回一个 URL 给 kubelet .这个 URL ,就是该 CRI shim 对应的 Streaming Server 的地址和端口.

4. kubelet 在拿到这个 URL 之后,就会把它以 Redirect 的方式返回给 API Server .

5. API Server 通过重定向来向 Streaming Server 发起真正的 /exec 请求,和它建立长连接.

6. Stream Server 这一部分具体怎么实现,完全可以由 CRI shim 的维护者自行决定.

### 参考文档
- https://juejin.im/post/5e1ec4456fb9a02ff254aaa7    //K8s、CRI与container
- https://www.cnblogs.com/xuxinkun/p/8036832.html    //docker、oci、runc以及kubernetes梳理
- https://www.jianshu.com/p/517e757d6d17             //深入理解容器基础概念
- https://feisky.gitbooks.io/kubernetes/plugins/CRI.html  //容器运行时文档(推荐阅读）
- https://www.cnblogs.com/zll-0405/p/10786541.html  //[Kubernetes] CRI 的设计与工作原理
- https://www.cnblogs.com/justinli/p/11578951.html   //kubernetes CRI 前世今生