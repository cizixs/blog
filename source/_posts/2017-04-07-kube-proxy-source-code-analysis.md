---
layout: post
title: "kube-proxy 源码解析"
excerpt: "`kube-proxy` 运行在 kubernetes 集群中每个 worker 节点上，负责实现 service 这个概念提供的功能。`kube-proxy` 会把访问 service VIP 的请求转发到运行的 pods 上，实现负载均衡。"
categories: blog
tags: [ kubernetes, container, network, golang, kube-proxy, iptables]
comments: true
share: true
---

## kube-proxy 功能简介

我们在之前的文章中[介绍过 kube-proxy 和 service](http://cizixs.com/2017/03/30/kubernetes-introduction-service-and-kube-proxy)的工作原理，以及它们的使用方法和功能。我们再来总结一下，`kube-proxy` 运行在 kubernetes 集群中每个 worker 节点上，负责实现 service 这个概念提供的功能。`kube-proxy` 会把访问 service VIP 的请求转发到运行的 pods 上，实现负载均衡。

![](https://ww3.sinaimg.cn/large/006tNc79gy1fee5voyscmj31hm0p0gnm.jpg)

当用户创建 service 的时候，endpointController 会根据 service 的 selector 找到对应的 pod，然后生成 endpoints 对象保存到 etcd 中。kube-proxy 的主要工作就是监听 etcd（通过 apiserver 的接口，而不是直接读取 etcd），来实时更新节点上的 iptables。

service 有关的信息保存在 etcd 的 `/registry/services` 目录，比如在我的集群中，这个目录的内容是这样的：

```bash
~]$ etcdctl ls --recursive  /registry/services
/registry/services/endpoints
/registry/services/endpoints/default
/registry/services/endpoints/default/whoami
/registry/services/endpoints/default/kubernetes
/registry/services/endpoints/kube-system
/registry/services/endpoints/kube-system/kube-controller-manager
/registry/services/endpoints/kube-system/container-log
/registry/services/endpoints/kube-system/container-terminal
/registry/services/endpoints/kube-system/kube-scheduler
/registry/services/endpoints/kube-system/kube-dns
/registry/services/specs
/registry/services/specs/default
/registry/services/specs/default/kubernetes
/registry/services/specs/default/whoami
/registry/services/specs/kube-system
/registry/services/specs/kube-system/kube-dns
/registry/services/specs/kube-system/container-log
/registry/services/specs/kube-system/container-terminal
```

这篇文章我们会分析 kube-proxy 的源码，讲解在 `iptables` 模式下它的原理。文章所有的代码基于 kubernetes 1.5.0，为了增强可读性会对某些部分做删减（错误处理、日志打印、无关的功能等）。

## 函数入口

和 kubernetes 其他组件一样，`kube-proxy` 入口在 `kubernetes/cmd/kube-proxy`，具体的代码如下：

```go
func main() {
	config := options.NewProxyConfig()
	config.AddFlags(pflag.CommandLine)
	flag.InitFlags()

    ...
	s, err := app.NewProxyServerDefault(config)
    ...

	if err = s.Run(); err != nil {
		fmt.Fprintf(os.Stderr, "%v\n", err)
		os.Exit(1)
	}
}
```

上述代码的大概过程是：用 `options.NewProxyConfig()` 创建出默认的配置选项，然后用命令行的参数更新配置的内容；然后 `app.NewProxyServerDefault(config)` 利用配置创建服务，最后运行创建的服务，一直保持运行状态。

## 服务创建

kube-proxy 入口很重要的一部分就是创建 ProxyServer，我们来看一下 `app.NewProxyServerDefault(config)` 内部的实现，这个函数定义是在 `cmd/kube-proxy/app/server.go`：

```go
func NewProxyServerDefault(config *options.ProxyServerConfig) (*ProxyServer, error) {
	...
	protocol := utiliptables.ProtocolIpv4
	if net.ParseIP(config.BindAddress).To4() == nil {
		protocol = utiliptables.ProtocolIpv6
	}

	var netshInterface utilnetsh.Interface
	var iptInterface utiliptables.Interface
	var dbus utildbus.Interface

	execer := exec.New()
	...
	// Create event recorder
	hostname := nodeutil.GetHostname(config.HostnameOverride)
	eventBroadcaster := record.NewBroadcaster()
	recorder := eventBroadcaster.NewRecorder(api.EventSource{Component: "kube-proxy", Host: hostname})

	var proxier proxy.ProxyProvider
	var endpointsHandler proxyconfig.EndpointsConfigHandler

	proxyMode := getProxyMode(string(config.Mode), client.Core().Nodes(), hostname, iptInterface, iptables.LinuxKernelCompatTester{})
	if proxyMode == proxyModeIPTables {
		if config.IPTablesMasqueradeBit == nil {
			return nil, fmt.Errorf("Unable to read IPTablesMasqueradeBit from config")
		}
		proxierIPTables, err := iptables.NewProxier(iptInterface, utilsysctl.New(), execer, config.IPTablesSyncPeriod.Duration, config.IPTablesMinSyncPeriod.Duration, config.MasqueradeAll, int(*config.IPTablesMasqueradeBit), config.ClusterCIDR, hostname, getNodeIP(client, hostname))

		proxier = proxierIPTables
		endpointsHandler = proxierIPTables
	    ...
	} else {
		glog.V(0).Info("Using userspace Proxier.")
		...
	}

	serviceConfig := proxyconfig.NewServiceConfig()
	serviceConfig.RegisterHandler(proxier)

	endpointsConfig := proxyconfig.NewEndpointsConfig()
	endpointsConfig.RegisterHandler(endpointsHandler)

	proxyconfig.NewSourceAPI(
		client.Core().RESTClient(),
		config.ConfigSyncPeriod,
		serviceConfig.Channel("api"),
		endpointsConfig.Channel("api"),
	)

	...
	conntracker := realConntracker{}

	return NewProxyServer(client, config, iptInterface, proxier, eventBroadcaster, recorder, conntracker, proxyMode)
}
```

最终 `NewProxyServer()` 比较简单，把所有传给它的参数作为结构体的内容返回，这些参数中的解释如下：

- `client`：连接到 kubernetes api server 的客户端对象
- `config`: proxyServer 配置对象，包含了所有的配置信息
- `iptInterface`: iptables 对象，运行执行所有的 iptables 操作
- `proxier`: proxier 是具体执行转发逻辑的对象，不管 userspace 模式还是 iptables 模式，都是一个 proxier 对象
- `eventBroadcaster`: 事件广播对象，把 kube-proxy 内部发生的事件发送到 apiserver
- `recorder`: 事件记录对象
- `conntracker`: connection track 有关的操作
- `proxyMode`: 代理的模式，iptables 还是 userspace

这里面有两个比较重要的变量：`proxier` 和 `endpointsHandler`，它们的值都是 `proxierIPTables`。

### ServiceConfig 和 endpointsConfig

serviceConfig 和 endpointsConfig 分别负责服务和端点相关内容的同步，它们的原理都是一样的。我们这里只分析 serviceConfig，它的代码在 `pkg/proxy/config/config.go` 文件中。serviceConfig 结构如下：

```go
type ServiceConfig struct {
	mux     *config.Mux
	bcaster *config.Broadcaster
	store   *serviceStore
}
```

它有三个结构：`mux` 是个汇聚器，可以把发送过来的更新统一保存到内部，`serviceStore` 保存了发生变化的 Service 对象，`Broadcaster` 在一旦有变化出现的时候通知对应的处理函数就行处理。

```go
func NewServiceConfig() *ServiceConfig {
	updates := make(chan struct{}, 1)
	store := &serviceStore{updates: updates, services: make(map[string]map[types.NamespacedName]api.Service)}
	mux := config.NewMux(store)
	bcaster := config.NewBroadcaster()
	go watchForUpdates(bcaster, store, updates)
	return &ServiceConfig{mux, bcaster, store}
}

func watchForUpdates(bcaster *config.Broadcaster, accessor config.Accessor, updates <-chan struct{}) {
	for true {
		<-updates
		bcaster.Notify(accessor.MergedState())
	}
}

// 注册 handler，有变化的时候会调用对应的 handler 进行处理
func (c *ServiceConfig) RegisterHandler(handler ServiceConfigHandler) {
	c.bcaster.Add(config.ListenerFunc(func(instance interface{}) {
		glog.V(3).Infof("Calling handler.OnServiceUpdate()")
		handler.OnServiceUpdate(instance.([]api.Service))
	}))
}
```

上面这段代码可以看到 `NewServiceConfig` 初始化的时候会在后台启动一个 goroutine 运行 `watchForUpdates`，这个 goroutine 不断循环的逻辑就是上面提到的：一旦 updates 可读（service 对象有变化），就调用 `bcaster.Notify()` 把变化的内容（通过 `accessor.MergedState()` 函数得到的结果）进行通知，最终会调用在内部注册的 handler 函数。

**NOTE**: 这个注册-触发的逻辑是通过 `Broadcaster` 实现的，对应的代码在 `pkg/util/config/config.go`，它提供了两个方法：`Add()` 是添加 handler，可以添加多个；`notify()` 会把参数交给所有的 handler 进行处理。

注册 handler 的步骤是 `serviceConfig.RegisterHandler(proxier)`，所以最终会调用 `proxier.OnServiceUpdate（）`方法，这个方法就是 `pkg/proxy/iptables/proxier.go:Proxier` 定义的，会实现最终 iptables 规则的刷新。

说道这里，还有两者问题需要解决：**变化的内容是怎么获得的？谁会往 `updates channel` 中写东西？**

第一个问题的答案在 `serviceStore.MergedState()` 方法中：

```go
func (s *serviceStore) MergedState() interface{} {
	s.serviceLock.RLock()
	defer s.serviceLock.RUnlock()
	services := make([]api.Service, 0)
	for _, sourceServices := range s.services {
		for _, value := range sourceServices {
			services = append(services, value)
		}
	}
	return services
}
```

`serviceStore` 结构体内部用一个嵌套字典 `map[string]map[types.NamespacedName]api.Service` 保存了所有的 Service 对象，这个嵌套字典外层的键是来源，内层是对应的服务名和服务对象。`MergedState` 方法删除了最外层的来源，返回所有的 Service 对象，也就是起到了汇聚不同来源 Service 的功能。至于这个字典是谁在什么时候写进去的，我们后面再说。

第二个问题，我们要看下面这段代码的实现了：

```go
proxyconfig.NewSourceAPI(
	client.Core().RESTClient(),
	config.ConfigSyncPeriod,
	serviceConfig.Channel("api"),
	endpointsConfig.Channel("api"),
)
```

`NewSourceAPI` 有四个参数，第一个参数是 RESTClient，用来从 apiserver 获取请求；后面两个参数分别是 service 和 endpoints 的 channel，读取的数据最终会发送到这里。我们还是来看一下 `serviceConfig.Channel()` 的代码：

```go
func (c *ServiceConfig) Channel(source string) chan ServiceUpdate {
	ch := c.mux.Channel(source)
	serviceCh := make(chan ServiceUpdate)
	go func() {
		for update := range serviceCh {
			ch <- update
		}
		close(ch)
	}()
	return serviceCh
}
```

这段代码创建了两个 channel：一个是在 `c.mux` 中创建的，用来汇聚所有的 service 对象，一个是新建的 `ServiceUpdate` channel，最终作为返回值。在后台启动一个参数，会把 `ServiceUpdate` channel 中的东西，持续不断地转送到 `c.mux` channel 中。也就是说，任何写到 `serviceConfig.Channel("api")` 的内容最终都会被 `c.mux` 调用 `serviceStore.Merge()`，这个方法会把 channel 中的更新保存到字典中 `map[string]map[types.NamespacedName]api.Service`。这个可以回答我们上面留下的疑问，`serviceConfig` 中的数据是谁写进去的。

```go
func (s *serviceStore) Merge(source string, change interface{}) error {
	s.serviceLock.Lock()
	services := s.services[source]
	if services == nil {
		services = make(map[types.NamespacedName]api.Service)
	}
	update := change.(ServiceUpdate)
	switch update.Op {
	case ADD:
		glog.V(5).Infof("Adding new service from source %s : %s", source, spew.Sdump(update.Services))
		for _, value := range update.Services {
			name := types.NamespacedName{Namespace: value.Namespace, Name: value.Name}
			services[name] = value
		}
	case REMOVE:
		glog.V(5).Infof("Removing a service %s", spew.Sdump(update))
		for _, value := range update.Services {
			name := types.NamespacedName{Namespace: value.Namespace, Name: value.Name}
			delete(services, name)
		}
	case SET:
		glog.V(5).Infof("Setting services %s", spew.Sdump(update))
		// Clear the old map entries by just creating a new map
		services = make(map[types.NamespacedName]api.Service)
		for _, value := range update.Services {
			name := types.NamespacedName{Namespace: value.Namespace, Name: value.Name}
			services[name] = value
		}
	default:
		glog.V(4).Infof("Received invalid update type: %s", spew.Sdump(update))
	}
	s.services[source] = services
	s.serviceLock.Unlock()
	if s.updates != nil {
		// Since we record the snapshot before sending this signal, it's
		// possible that the consumer ends up performing an extra update.
		select {
		case s.updates <- struct{}{}:
		default:
			glog.V(4).Infof("Service handler already has a pending interrupt.")
		}
	}
	return nil
}
```

另外，在 `ServiceStore` 的最后，还会往 `s.updates` 写入一个数据，告诉监听在 channel 另一端说有数据更新，你可以调用处理函数来同步 iptables 了。

整个 `serviceConfig` 的逻辑是这样的：

![](https://ww4.sinaimg.cn/large/006tNc79gy1fee66e8rbbj30wy0io40h.jpg)

它对外暴露了一个 channel，任何写到这个 channel 中的数据，都会被 `mux` 自动保存到内部的 `serviceStore` 中，并往 `updates channel` 发一个通过，监听在 `updates channel` 另一端的 `bcaster` 接到通知，就调用处理函数 `proxier.OnServiceUpdate()` 去处理。

不难猜测，在 `NewSourceAPI` 函数的内部一定会有把读取的数据写到 `serviceConfig.Channel("api")` 的逻辑。

**NOTE:** endpointsConfig 和 serviceConfig 的实现原理完全一样，只不过监听的对象是 etcd 中的 endpoints，而不是 services。

### NewSourceAPI 和数据的真实来源

上面这部分我们看到了如果监听到的对象有变化，会执行对应的 iptables 同步处理函数，这部分我们讲讲 kube-proxy 是怎么监听 apiserver 的数据，并把结果转换成正确的格式的。

继续看 `NewSourceAPI` 的代码：

```go
func NewSourceAPI(c cache.Getter, period time.Duration, servicesChan chan<- ServiceUpdate, endpointsChan chan<- EndpointsUpdate) {
	servicesLW := cache.NewListWatchFromClient(c, "services", api.NamespaceAll, fields.Everything())
	cache.NewReflector(servicesLW, &api.Service{}, NewServiceStore(nil, servicesChan), period).Run()

	endpointsLW := cache.NewListWatchFromClient(c, "endpoints", api.NamespaceAll, fields.Everything())
	cache.NewReflector(endpointsLW, &api.Endpoints{}, NewEndpointsStore(nil, endpointsChan), period).Run()
}
```

`NewServiceAPI` 为 service 和 endpoints 分别创建了 `ListWatch` 和 `Reflector`，根据惯例，我们还是只分析 `Service` 有关的部分。

`cache.NewListWatchFromClient()` 方法定义在 `pkg/client/cache/listwatch.go` 文件，它主要的功能是从 apiserver 读取（list）和监听（watch）某个 uri 地址的数据。它的实现不是本文的重点，在此略过不表，有兴趣可以自行阅读，并不是很复杂。

`NewServiceStore` 是什么的呢？

```go
func NewServiceStore(store cache.Store, ch chan<- ServiceUpdate) cache.Store {
	fn := func(objs []interface{}) {
		var services []api.Service
		for _, o := range objs {
			services = append(services, *(o.(*api.Service)))
		}
		ch <- ServiceUpdate{Op: SET, Services: services}
	}
	if store == nil {
		store = cache.NewStore(cache.MetaNamespaceKeyFunc)
	}
	return &cache.UndeltaStore{
		Store:    store,
		PushFunc: fn,
	}
}
```

`Reflector` 会从 `ListWatcher` 中读取要监听资源的变化，写到 `store` 对象中去。这部分的代码在 `pkg/client/cache/` ，不是本文的重点。简单说一下它的大致过程：它内部进入循环，调用 `servicesLW.wathc()` 方法，根据得到的数据更新 `serviceStore` 的值，这个 serviceStore 每次更新都会出发一个 `pushFunc` 把当前的数据写到 channel `ServiceUpdate` ，从而完成了和上面部分的对接！

![](https://ww4.sinaimg.cn/large/006tNc79gy1fee6z9jczrj30sg0ig75i.jpg)

### OnServiceUpdate：最终干活的人

前面说了数据是怎么从 api Server 被 kube-rpoxy 拿到，并一层层地传递的。最终拿到了 service 和 endpoints 的数据，最终还是要落到谁来处理这些数据。前面我们也提过，不管是 `iptables` 模式，还是 `userspace` 模式，最终的处理函数都是 `OnServiceUpdate()`，我们这里还是看一下 `iptables/proxier.go:OnServiceUpdate`：

```go
func (proxier *Proxier) OnServiceUpdate(allServices []api.Service) {
	proxier.mu.Lock()
	defer proxier.mu.Unlock()
	proxier.haveReceivedServiceUpdate = true

	activeServices := make(map[proxy.ServicePortName]bool) // use a map as a set

	for i := range allServices {
		service := &allServices[i]
		svcName := types.NamespacedName{
			Namespace: service.Namespace,
			Name:      service.Name,
		}

		for i := range service.Spec.Ports {
			servicePort := &service.Spec.Ports[i]

			serviceName := proxy.ServicePortName{
				NamespacedName: svcName,
				Port:           servicePort.Name,
			}
			activeServices[serviceName] = true
			info, exists := proxier.serviceMap[serviceName]
			if exists && proxier.sameConfig(info, service, servicePort) {
				continue
			}
			if exists {
				glog.V(3).Infof("Something changed for service %q: removing it", serviceName)
				delete(proxier.serviceMap, serviceName)
			}
			serviceIP := net.ParseIP(service.Spec.ClusterIP)
			glog.V(1).Infof("Adding new service %q at %s:%d/%s", serviceName, serviceIP, servicePort.Port, servicePort.Protocol)
			info = newServiceInfo(serviceName)
			info.clusterIP = serviceIP
			info.port = int(servicePort.Port)
			info.protocol = servicePort.Protocol
			info.nodePort = int(servicePort.NodePort)
			info.externalIPs = service.Spec.ExternalIPs
			// Deep-copy in case the service instance changes
			info.loadBalancerStatus = *api.LoadBalancerStatusDeepCopy(&service.Status.LoadBalancer)
			info.sessionAffinityType = service.Spec.SessionAffinity
			info.loadBalancerSourceRanges = service.Spec.LoadBalancerSourceRanges

			proxier.serviceMap[serviceName] = info
			glog.V(4).Infof("added serviceInfo(%s): %s", serviceName, spew.Sdump(info))
		}
	}

	staleUDPServices := sets.NewString()
	// Remove serviceports missing from the update.
	for name, info := range proxier.serviceMap {
		if !activeServices[name] {
			glog.V(1).Infof("Removing service %q", name)
			if info.protocol == api.ProtocolUDP {
				staleUDPServices.Insert(info.clusterIP.String())
			}
			delete(proxier.serviceMap, name)
		}
	}
	proxier.syncProxyRules()
	proxier.deleteServiceConnections(staleUDPServices.List())
}
```

这段代码的核心是遍历作为参数传进来的 `api.Service` 数组，根据里面的信息构建 `proxier.serviceMap`（service 有改动，或者新建、删除），然后调用 `proxier.syncProxyRules()` 去同步 iptables 规则列表。

通过这个部分的分析，我们明白了 `kube-proxy` 是如何保证 `apiserver` 中数据一旦变化，就立即更新节点的 iptables 规则的。

这个过程的流程图如下：

![](https://ww3.sinaimg.cn/large/006tNc79gy1fee5sbtsraj31fy0oggnx.jpg)

## 服务的运行

 `ProxyServer` 初始化结束之后，还会调用 `s.Run()`，我们来看一下里面的内容：

```
func (s *ProxyServer) Run() error {

	// Start up a webserver if requested
	if s.Config.HealthzPort > 0 {
		http.HandleFunc("/proxyMode", func(w http.ResponseWriter, r *http.Request) {
			fmt.Fprintf(w, "%s", s.ProxyMode)
		})
		configz.InstallHandler(http.DefaultServeMux)
		go wait.Until(func() {
			err := http.ListenAndServe(s.Config.HealthzBindAddress+":"+strconv.Itoa(int(s.Config.HealthzPort)), nil)
			if err != nil {
				glog.Errorf("Starting health server failed: %v", err)
			}
		}, 5*time.Second, wait.NeverStop)
	}

	// Tune conntrack, if requested
	if s.Conntracker != nil && runtime.GOOS != "windows" {
		max, err := getConntrackMax(s.Config)
		if err != nil {
			return err
		}
		if max > 0 {
			err := s.Conntracker.SetMax(max)
			if err != nil {
				if err != readOnlySysFSError {
					return err
				}
				const message = "DOCKER RESTART NEEDED (docker issue #24000): /sys is read-only: " +
					"cannot modify conntrack limits, problems may arise later."
				s.Recorder.Eventf(s.Config.NodeRef, api.EventTypeWarning, err.Error(), message)
			}
		}

		if s.Config.ConntrackTCPEstablishedTimeout.Duration > 0 {
			timeout := int(s.Config.ConntrackTCPEstablishedTimeout.Duration / time.Second)
			if err := s.Conntracker.SetTCPEstablishedTimeout(timeout); err != nil {
				return err
			}
		}

		if s.Config.ConntrackTCPCloseWaitTimeout.Duration > 0 {
			timeout := int(s.Config.ConntrackTCPCloseWaitTimeout.Duration / time.Second)
			if err := s.Conntracker.SetTCPCloseWaitTimeout(timeout); err != nil {
				return err
			}
		}
	}

	// Birth Cry after the birth is successful
	s.birthCry()

	// Just loop forever for now...
	s.Proxier.SyncLoop()
	return nil
}
```

这是个会一直运行的函数，前面的部分主要给 kube-proxy 做一些额外的补充，比如启动 `healthz` web 服务器，优化 `conntrack` 的参数。最后会调用 `s.Proxier.SyncLoop()` 进入主循环。

```go
func (proxier *Proxier) Sync() {
	proxier.mu.Lock()
	defer proxier.mu.Unlock()
	proxier.syncProxyRules()
}

func (proxier *Proxier) SyncLoop() {
	t := time.NewTicker(proxier.syncPeriod)
	defer t.Stop()
	for {
		<-t.C
		glog.V(6).Infof("Periodic sync")
		proxier.Sync()
	}
}
```

这是一个周期性的任务，每隔  `proxier.syncPeriod` 的时间周期（默认是 30s，可以通过启动参数 `--iptables-sync-period` 配置）就会调用 `proxier.syncProxyRules()` 对 iptables 进行更新。

这里有个疑问：既然 `kube-proxy` 能够自动监听 apiserver 的变化，并更新 iptables，为什么这里还要再每隔一段时间强制同步一次呢？我的看法是这只是安全防护措施，来规避有些情况（比如代码 bug，或者网络、环境问题等原因）下数据可能没有及时同步。

## 参考资料

- [kube-proxy 源码解析](http://licyhust.com/%E5%AE%B9%E5%99%A8%E6%8A%80%E6%9C%AF/2016/11/05/kube-proxy/)
- [CSDN 博客：kube-proxy源码分析](http://blog.csdn.net/waltonwang/article/details/55286724)
