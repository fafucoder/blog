---
title: kubernetes自定义CRD
date: 2020-01-31 13:08:29
tags:
- kubernetes
- crd
categories:
- kubernetes
---

### CRD简介
K8S中一切都是resource，比如Deployment，Service等等。

我们可以基于CRD（CustomResourceDefinitions）功能新增resource，比如我想自定义一种Deployment资源，提供不同的部署策略。

k8s中resource可以通过RESTFUL API进行CURD操作，对于CRD创建的resource也是一样的。

CRD仅仅是定义一种resource，我们还需要实现controller，类似于deployment controller等等，监听对应资源的CURD事件，做出对应的处理，比如部署POD。

[CRD官方文档](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions)


### 编写自定义CRD
编写自定义CRD包括两部分， 一个是定义crd, 一个是编写controller

####自定义CRD
```yaml
apiVersion: apiextensions.k8s.io/v1   #版本
kind: CustomResourceDefinition        #类型
metadata:
  name: crontabs.stable.example.com   #名称必须符合下面的格式：<plural>.<group>
spec:
  group: stable.example.com           # REST API使用的组名称：/apis/<group>/<version>
  version: v1  #版本
  scope: Namespaced   #Namespaced或Cluster
  names:
    plural: crontabs  #复数名， URL中使用的复数名称: /apis/<group>/<version>/<plural>
    singular: crontab  #单数名
    kind: CronTab   # CamelCased格式的单数类型。在清单文件中使用
    shortNames:   # CLI中使用的资源简称
    - ct
  additionalPrinterColumns:  #终端显示额外显示的信息
    - name: Spec
      type: string
      description: The cron spec defining the interval a CronJob is run
      JSONPath: .spec.cronSpec
    - name: Replicas
      type: integer
      description: The number of jobs launched by the CronJob
      JSONPath: .spec.replicas
    - name: Age
      type: date
      JSONPath: .metadata.creationTimestamp
  validation: #验证规则
    openAPIV3Schema:
      properties:
        spec:
          properties:
            cronSpec: #--必须是字符串，并且必须是正则表达式所描述的形式
              type: string
              pattern: '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
            replicas: #----必须是整数，最小值必须为1，最大值必须为10
              type: integer
              minimum: 1
              maximum: 10
```
自定义crd还可以包含自定义验证规则,额外打印信息等

编写完自定义crd, 只需`kubectl apply -f xxx.yaml`就行

定义完crd就可以定义cr

#### 定义CR
```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
```
通过`kubectl create -f crd.yaml`可以创建一个CRD

通过`kubectl get crd`可以获取创建的所有CRD。

#### 定义controller
定义完crd部分， 就需要实现controller部分

可以使用client-go来作为Kubernetes的客户端， 常用的controller生成框架包括[Kubebuilder](https://github.com/kubernetes-sigs/kubebuilder)

官方的自定义controller例子[sample-controller](https://github.com/kubernetes/sample-controller)

#### client-go原理
client-go的原理图如下：

![client原理图](http://maoqide.live/media/posts/cloud/sample-controller/client-go-controller-interaction.jpeg)

具体流程就是list-watch机制， api-server会发送指定事件， controller通过watch事件， 处理事件。

各名词解析：

Reflector: 定义在 cache 包的 Reflector 类中，它监听特定资源类型(Kind)的 Kubernetes API，在ListAndWatch方法中执行。监听的对象可以是 Kubernetes 的内置资源类型或者是自定义资源类型。当 reflector 通过 watch API 发现新的资源实例被创建，它将通过对应的 list API 获取到新创建的对象并在watchHandler方法中将其加入到Delta Fifo队列中。

Informer: 定义在 cache 包的 base controller 中，它从Delta Fifo队列中 pop 出对象，在processLoop方法中执行。base controller 的工作是将对象保存一遍后续获取，并调用 controller 将对象传给 controller。

Indexer: 提供对象的 indexing 方法，定义在 cache 包的 Indexer中。一个典型的 indexing 的应用场景是基于对象的 label 创建索引。Indexer 基于几个 indexing 方法维护索引，它使用线程安全的 data store 来存储对象和他们的key。在 cache 包的 Store 类中定义了一个名为MetaNamespaceKeyFunc的默认方法，可以为对象生成一个<namespace>/<name>形式的key。

#### 编写controller代码
这部分代码来源于[maoqide](http://maoqide.live/post/cloud/sample-controller/)
```golang
/*
*** main.go
*/
// 创建 clientset
kubeClient, err := kubernetes.NewForConfig(cfg)		// k8s clientset, "k8s.io/client-go/kubernetes"
exampleClient, err := clientset.NewForConfig(cfg)	// sample clientset, "k8s.io/sample-controller/pkg/generated/clientset/versioned"

// 创建 Informer
kubeInformerFactory := kubeinformers.NewSharedInformerFactory(kubeClient, time.Second*30)		// k8s informer, "k8s.io/client-go/informers"
exampleInformerFactory := informers.NewSharedInformerFactory(exampleClient, time.Second*30)		// sample informer, "k8s.io/sample-controller/pkg/generated/informers/externalversions"

// 创建 controller，传入 clientset 和 informer
controller := NewController(kubeClient, exampleClient,
		kubeInformerFactory.Apps().V1().Deployments(),
		exampleInformerFactory.Samplecontroller().V1alpha1().Foos())

// 运行 Informer，Start 方法为非阻塞，会运行在单独的 goroutine 中
kubeInformerFactory.Start(stopCh)	
exampleInformerFactory.Start(stopCh)

// 运行 controller
controller.Run(2, stopCh)

/*
*** controller.go 
*/
NewController() *Controller {
	// 将 CRD 资源类型定义加入到 Kubernetes 的 Scheme 中，以便 Events 可以记录 CRD 的事件
	utilruntime.Must(samplescheme.AddToScheme(scheme.Scheme))

	// 创建 Broadcaster
	eventBroadcaster := record.NewBroadcaster()
	// ... ...

	// 监听 CRD 类型'Foo'并注册 ResourceEventHandler 方法，当'Foo'的实例变化时进行处理
	fooInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: controller.enqueueFoo,
		UpdateFunc: func(old, new interface{}) {
			controller.enqueueFoo(new)
		},
	})

	// 监听 Deployment 变化并注册 ResourceEventHandler 方法，
	// 当它的 ownerReferences 为 Foo 类型实例时，将该 Foo 资源加入 work queue
	deploymentInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: controller.handleObject,
		UpdateFunc: func(old, new interface{}) {
			newDepl := new.(*appsv1.Deployment)
			oldDepl := old.(*appsv1.Deployment)
			if newDepl.ResourceVersion == oldDepl.ResourceVersion {
				return
			}
			controller.handleObject(new)
		},
		DeleteFunc: controller.handleObject,
	})
}

func (c *Controller) Run(threadiness int, stopCh <-chan struct{}) error {
	// 在启动 worker 前等待缓存同步
	if ok := cache.WaitForCacheSync(stopCh, c.deploymentsSynced, c.foosSynced); !ok {
		return fmt.Errorf("failed to wait for caches to sync")
	}

	// 运行两个 worker 来处理资源
	for i := 0; i < threadiness; i++ {
		go wait.Until(c.runWorker, time.Second, stopCh)
	}
}

// 无限循环，不断的调用 processNextWorkItem 处理下一个对象
func (c *Controller) runWorker() {
    for c.processNextWorkItem() {
    }
}

// 从workqueue中获取下一个对象并进行处理，通过调用 syncHandler
func (c *Controller) processNextWorkItem() bool {
    obj, shutdown := c.workqueue.Get()
    if shutdown {
        return false
    }

    err := func(obj interface{}) error {
        // 调用 workqueue.Done(obj) 方法告诉 workqueue 当前项已经处理完毕，
        // 如果我们不想让当前项重新入队，一定要调用 workqueue.Forget(obj)。
        // 当我们没有调用Forget时，当前项会重新入队 workqueue 并在一段时间后重新被获取。
        defer c.workqueue.Done(obj)
        var key string
        var ok bool

        // 我们期望的是 key 'namespace/name' 格式的 string
        if key, ok = obj.(string); !ok {
            // 无效的项调用Forget方法，避免重新入队。
            c.workqueue.Forget(obj)
            utilruntime.HandleError(fmt.Errorf("expected string in workqueue but got %#v", obj))
            return nil
        }

        if err := c.syncHandler(key); err != nil {
            // 放回workqueue避免偶发的异常
            c.workqueue.AddRateLimited(key)
            return fmt.Errorf("error syncing '%s': %s, requeuing", key, err.Error())
        }

        // 如果没有异常，Forget当前项，同步成功
        c.workqueue.Forget(obj)
        klog.Infof("Successfully synced '%s'", key)
        return nil
    }(obj)

    if err != nil {
        utilruntime.HandleError(err)
        return true
    }

    return true
}

// 对比真实的状态和期望的状态并尝试合并，然后更新Foo类型实例的状态信息
func (c *Controller) syncHandler(key string) error {
    // 通过 workqueue 中的 key 解析出 namespace 和 name
    namespace, name, err := cache.SplitMetaNamespaceKey(key)
    // 调用 lister 接口通过 namespace 和 name 获取 Foo 实例
    foo, err := c.foosLister.Foos(namespace).Get(name)
    deploymentName := foo.Spec.DeploymentName
    // 获取 Foo 实例中定义的 deploymentname
    deployment, err := c.deploymentsLister.Deployments(foo.Namespace).Get(deploymentName)

    // 没有发现对应的 deployment，新建一个
    if errors.IsNotFound(err) {
        deployment, err = c.kubeclientset.AppsV1().Deployments(foo.Namespace).Create(newDeployment(foo))
    }

    // OwnerReferences 不是 Foo 实例，warning并返回错误
    if !metav1.IsControlledBy(deployment, foo) {
        msg := fmt.Sprintf(MessageResourceExists, deployment.Name)
        c.recorder.Event(foo, corev1.EventTypeWarning, ErrResourceExists, msg)
        return fmt.Errorf(msg)
    }

    // deployment 中 的配置和 Foo 实例中 Spec 的配置不一致，即更新 deployment
    if foo.Spec.Replicas != nil && *foo.Spec.Replicas != *deployment.Spec.Replicas {
        deployment, err = c.kubeclientset.AppsV1().Deployments(foo.Namespace).Update(newDeployment(foo))
    }

    // 更新 Foo 实例状态
    err = c.updateFooStatus(foo, deployment)
    c.recorder.Event(foo, corev1.EventTypeNormal, SuccessSynced, MessageResourceSynced)
}
```

### 参考文档
- https://blog.csdn.net/boling_cavalry/article/details/88934063  //编写自定义CRD三部曲， 跟着编写就能实现一个自定义CRD
- http://maoqide.live/post/cloud/sample-controller/  //CRD的实现原理， 以及简要代码
- https://jimmysong.io/kubernetes-handbook/concepts/crd.html   //CRD的文档
- https://www.jianshu.com/p/cc7eea6dd1fb  //如何定义一个crd
