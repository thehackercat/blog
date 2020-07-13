---
title: 浅谈 k8s service&kube-proxy
date: 2018-09-14 19:10:46
comment: true
weight: 5
draft: false
categories: ["Code", "Kubernetes"]
tags: ["Golang", "Kubernetes", "Linux"]
lightgallery: true
thumbnail: http://fullbit.ca/wp-content/uploads/2018/02/kubernetes-logo1-e1525258419775.png
---
<div style="text-align: right;">Lexus Lee</div>

### 背景

最开始听到同事 k8s 分享时比较困惑我的一个问题是 k8s 怎么实现一个私有 ip(虚拟 ip，以下简称 vip)到另一个私有ip收发包的。

不过其实我想知道的应该是 k8s 通信机制，它是怎么实现服务发现的，新建的 pod 是怎么感知到的，万一有些 pod 节点变更 vip 变了 k8s 是如何感知的。

基于这个问题，做一下关于 k8s service&kube-proxy 的分享。
<!--more-->


### Service&kube-proxy 概述

首先我建了一个 `replcas = 4`  `lebel: app=service_test_pod`的 python server deployment 来打出当前 pod 的 Hostname

``` yaml
apiVersion:  apps/v1
kind: Deployment
metadata:
  name: service-test
spec:
  replicas: 4
  selector:
    matchLabels:
      app: service_test_pod
  template:
    metadata:
      labels:
        app: service_test_pod
    spec:
      containers:
      - name: simple-http
        image: python:2.7
        imagePullPolicy: IfNotPresent
        command: ["/bin/bash"]
        args: ["-c", "echo \"<p>Hello from $(hostname)</p>\" > index.html; python -m SimpleHTTPServer 9999"]
        ports:
        - name: http
          containerPort: 9999
```

可以看到启动了4个 pod

```
service-test-69764ddb4c-4hr2x                                     1/1       Running             0          8s        172.18.234.21    brand6
service-test-69764ddb4c-gstft                                     1/1       Running             0          8s        172.18.83.225    belba2
service-test-69764ddb4c-nnv29                                     1/1       Running             0          8s        172.18.156.140   brand2
service-test-69764ddb4c-vx5pn                                     0/1       ContainerCreating   0          8s        <none>           belba3
```

我必须挨个 curl 才能得到他们的 hostname

```shell
lilingzhi@belba1 ~/k8s/test $ curl 172.18.234.21:9999
<p>Hello from service-test-69764ddb4c-4hr2x</p>
```

但不能让其他 pod 直接通过 vip 访问这些 pod ，需要一个更上层的一个抽象，把这4个提供相同服务的 pod 打包成一个对外的服务，通过某个入口地址来访问它，并且把请求均衡到4个pod上，这样一层的抽象包装是实现一个服务网络(service mesh)的基础。

而 k8s service 就是做这个的。

首先我们来看下什么是 k8s service:

> - A Kubernetes `Service` is an abstraction which defines a logical set of `Pods` and a policy by which to access them - sometimes called a micro-service. The set of `Pods` targeted by a `Service` is (usually) determined by a [`Label Selector`](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors) (see below for why you might want a `Service` without a selector)

而打包pod成service并发布的微服务得支持 k8s 内部及外部的访问。

故 k8s service 提供了以下三种暴露 service 入口的模式:

> - ClusterIP: use a cluster-internal IP only - this is the default and is discussed above. Choosing this value means that you want this service to be reachable only from inside of the cluster.
> - NodePort: on top of having a cluster-internal IP, expose the service on a port on each node of the cluster (the same port on each node). You’ll be able to contact the service on any :NodePort address.
> - LoadBalancer: on top of having a cluster-internal IP and exposing service on a NodePort also, ask the cloud provider for a load balancer which forwards to the Service exposed as a :NodePort for each Node.

可以简单理解为 ClusterIp 是提供对内的访问入口，NodePort 和 LoadBalancer 是提供对外的，不过 LoadBalancer 是在暴露 NodePort 基础上提供可以接入外部的 LB。

那么我新建一个 service 给刚刚的 4 个 pod

```shell
lilingzhi@belba1 ~/k8s/test $ kubectl expose deployment service-test --type="NodePort" --port 9098 --target-port=9999
service/service-test exposed
```

可以看到

```shell
lilingzhi@belba1 ~/k8s/test $ kubectl get service -o wide
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                               AGE       SELECTOR
service-test           NodePort    172.19.97.3     <none>        9098:30255/TCP                        16s       app=service_test_pod
```

`service-test` 这儿就映射到 clusterIp 的 172.19.97.3:9098 端口上

再看下 endpoints

```shell
lilingzhi@belba1 ~/k8s/test $ kubectl get endpoints -o wide
NAME                   ENDPOINTS                                                               AGE
service-test           172.18.156.140:9999,172.18.193.66:9999,172.18.234.21:9999 + 1 more...   3m
```

可以看到也建了一个包含 4个 host:port 的元组的 endpoint

通过不断 curl service ip:port 会发现请求已经均衡到4个 pod 上了

```shell
lilingzhi@belba1 ~/k8s/test $ curl 172.19.97.3:9098
<p>Hello from service-test-69764ddb4c-4hr2x</p>
lilingzhi@belba1 ~/k8s/test $ curl 172.19.97.3:9098
<p>Hello from service-test-69764ddb4c-gstft</p>
lilingzhi@belba1 ~/k8s/test $ curl 172.19.97.3:9098
<p>Hello from service-test-69764ddb4c-4hr2x</p>
lilingzhi@belba1 ~/k8s/test $ curl 172.19.97.3:9098
<p>Hello from service-test-69764ddb4c-gstft</p>
```

由于 k8s service 路由是通过 kube-proxy 决定的，默认是走的 iptables 转发的(可换成用户态 proxy 或 ipvs)，所以查一下相应的 iptables 规则

```shell
$ sudo iptables -L -v -n -t nat

Chain KUBE-SERVICES (2 references)
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !172.18.0.0/16        172.19.97.3          /* default/service-test: cluster IP */ tcp dpt:9098
    0     0 KUBE-SVC-LY73ZDGF4KGO4YFJ  tcp  --  *      *       0.0.0.0/0            172.19.97.3          /* default/service-test: cluster IP */ tcp dpt:9098

```

看到有条 `chain KUBE-SVC-LY73ZDGF4KGO4YFJ` 定义了 service-test 的转发规则，于是查看相应的 chain

```shell
Chain KUBE-SVC-LY73ZDGF4KGO4YFJ (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-SEP-2T6K76SEPIPV3QKW  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/service-test: */ statistic mode random probability 0.25000000000
    0     0 KUBE-SEP-75XULILUFIHXBBLY  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/service-test: */ statistic mode random probability 0.33332999982
    0     0 KUBE-SEP-FFUABGNOKSNXGH7E  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/service-test: */ statistic mode random probability 0.50000000000
    0     0 KUBE-SEP-MKK3T4CLASKIDMJS  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/service-test: */
```

可以看到这条 `KUBE-SVC-LY73ZDGF4KGO4YFJ` chain 里有4个下一跳的 chain `KUBE-SEP-2T6K76SEPIPV3QKW` ，`KUBE-SEP-75XULILUFIHXBBLY`, `KUBE-SEP-FFUABGNOKSNXGH7E`, `KUBE-SEP-MKK3T4CLASKIDMJS`, 他们分别对应4个 pod 的 vip，并且设置了 iptables probability, 由上至下分别为 `25%`, `33.33%`, `50%`, `100%`, 由于 iptables 是顺序读的，这样确保了每个 pod 都是 `25%` 的请求分发的机率。

那么我们挑其中一条子 chain `KUBE-SEP-2T6K76SEPIPV3QKW` 跟进去看看

```shell
Chain KUBE-SEP-2T6K76SEPIPV3QKW (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       172.18.156.140       0.0.0.0/0            /* default/service-test: */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/service-test: */ tcp to:172.18.156.140:9999
```

果不其然，这儿的 source ip 就是对应 pod 的 vip, 一个请求就这样转发到 pod 上，可以看到一共要经过3层 chain。

![chains](http://7xse6j.com1.z0.glb.clouddn.com/k8s_share2.png)

不过有二义性的地方在于，这儿的 kube-proxy 实际上并不起一个 proxy 的作用，而是 watch 变更并更新 iptables，也就是说，client 的请求直接通过 iptables 路由，所以如果我们直接修改 iptables 也是可以奏效的。

### kube-proxy 源码分析

因为 kube-proxy 源码相对比较少，所以读了下源码，但还是蛮复杂的

kube-proxy 会作为 daemon 跑在每个节点上，对 api-server 中的 service & endpoint 进行 watch ,一旦检测到更新则往 iptables 里全量推送新的转发规则，那么我们根据 `kubernetes/cmd/kube-proxy/proxy.go` 里找到 kube-proxy 真正的入口函数 `Run()`

```go
func (s *ProxyServer) Run() error {
    ...
	serviceConfig := config.NewServiceConfig(informerFactory.Core().V1().Services(), s.ConfigSyncPeriod)
	serviceConfig.RegisterEventHandler(s.ServiceEventHandler)
	go serviceConfig.Run(wait.NeverStop)

	endpointsConfig := config.NewEndpointsConfig(informerFactory.Core().V1().Endpoints(), s.ConfigSyncPeriod)
	endpointsConfig.RegisterEventHandler(s.EndpointsEventHandler)
	go endpointsConfig.Run(wait.NeverStop)

	// This has to start after the calls to NewServiceConfig and NewEndpointsConfig because those
	// functions must configure their shared informer event handlers first.
	go informerFactory.Start(wait.NeverStop)

	// Birth Cry after the birth is successful
	s.birthCry()

	// Just loop forever for now...
	s.Proxier.SyncLoop()
    return nil
}
```

这就是 kube-proxy 的入口函数，实际上即起一个 proxy server daemon

可以看到通过 `informerFactory `新建了2个 config 对象(serviceConfig, endpointsConfig), 这个 `informerFactory` 是什么呢？

k8s 里所有资源都存在 etcd 中提供 api 通过 apiserver 的接口访问，其中有个核心的公共组件即 `informer` 是对 apiserver 资源访问的一层包装，其中包括 api 访问, localcache 等...

所以这里的两个 config 对象即用来获取 etcd 中 service 和 endpoints 的信息，他们都调用了 `RegisterEventHandler` 注册了一个回调函数，这个函数用来监听变更并发送变更信号。

最后用 goroutine 跑起来。

之后的 `informerFactory.Start()` 则用来初始化 informer 对象注册的回调函数。

最后把 proxy server loop 跑起来 `Proxier.SyncLoop()` 等待信号。



接着我们先看下 service 的  `NewServiceConfig` 这个函数，因为 endpoint 估计也是类似的。

```go
// NewServiceConfig creates a new ServiceConfig.
func NewServiceConfig(serviceInformer coreinformers.ServiceInformer, resyncPeriod time.Duration) *ServiceConfig {
	result := &ServiceConfig{
		lister:       serviceInformer.Lister(),
		listerSynced: serviceInformer.Informer().HasSynced,
	}

	serviceInformer.Informer().AddEventHandlerWithResyncPeriod(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    result.handleAddService,
			UpdateFunc: result.handleUpdateService,
			DeleteFunc: result.handleDeleteService,
		},
		resyncPeriod,
	)

	return result
}
```

看到它结构体里就两个对象 `lister` 和 `listerSynced`，所以我们接着看下 `serviceInformer`

```go
type ServiceInformer interface {
	Informer() cache.SharedIndexInformer
	Lister() v1.ServiceLister
}
```

看到是个通用对象 `SharedIndexInformer` 即上述提及的公共组件，故不往里深究。

接着回来看注册进去的回调函数，举个栗子，这儿的 `UpdateFunc: result.handleUpdateService` 最终会调到 `iptables.go` 里的 `OnServiceUpdate`

```go
func (c *ServiceConfig) handleUpdateService(oldObj, newObj interface{}) {
	oldService, ok := oldObj.(*v1.Service)
	if !ok {
		utilruntime.HandleError(fmt.Errorf("unexpected object type: %v", oldObj))
		return
	}
	service, ok := newObj.(*v1.Service)
	if !ok {
		utilruntime.HandleError(fmt.Errorf("unexpected object type: %v", newObj))
		return
	}
	for i := range c.eventHandlers {
		glog.V(4).Infof("Calling handler.OnServiceUpdate")
		c.eventHandlers[i].OnServiceUpdate(oldService, service)
	}
}


func (proxier *Proxier) OnServiceUpdate(oldService, service *v1.Service) {
	if proxier.serviceChanges.Update(oldService, service) && proxier.isInitialized() {
		proxier.syncRunner.Run()
	}
}
```

所以监听到变更之后调用的这个 `proxier.syncRunner.Run()` 是什么呢，得看下这个 `syncRunner` 在做什么

```go
proxier.syncRunner = async.NewBoundedFrequencyRunner("sync-runner", proxier.syncProxyRules, minSyncPeriod, syncPeriod, burstSyncs)

func (bfr *BoundedFrequencyRunner) Run() {
	// If it takes a lot of time to run the underlying function, noone is really
	// processing elements from <run> channel. So to avoid blocking here on the
	// putting element to it, we simply skip it if there is already an element
	// in it.
	select {
	case bfr.run <- struct{}{}:
	default:
	}
}

func (bfr *BoundedFrequencyRunner) Loop(stop <-chan struct{}) {
	glog.V(3).Infof("%s Loop running", bfr.name)
	bfr.timer.Reset(bfr.maxInterval)
	for {
		select {
		case <-stop:
			bfr.stop()
			glog.V(3).Infof("%s Loop stopping", bfr.name)
			return
		case <-bfr.timer.C():
			bfr.tryRun()
		case <-bfr.run:
			bfr.tryRun()
		}
	}
}
```

`syncRunner` 里注册了个 `syncProxyRules` 的回调函数，而刚刚 updateSync 中触发的 run 函数则用 select 发送了一个 `bfr.run` 信号，之前所提及的 proxy server 一旦收到信号就会跑一次 `tryRun()`调用到这个 `syncProxyRules` 的回调函数。

于是我们走到 `syncProxyRules`, 这个函数特别长，但简而言之就是做一些 iptables 的 ensure 和 update 操作。

这样一来，整个流程就走通了。

靠北，开发 k8s 的 google 工程师真的很机车，代码真的有点绕哦！

### 最后

最后我们来理一下 service && endpoint && kube-proxy 的关系。

service / endpoint 是pod对外暴露访问地址的封装，Kube-proxy 用来管理这些封装，做一些 ensure&update 的操作

![kube-proxy model](http://7xse6j.com1.z0.glb.clouddn.com/k8s_share.png)
