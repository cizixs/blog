---
layout: post
title: "kubelet 源码分析：启动流程"
excerpt: "kubelet 的核心工作是监听 apiserver，一旦发现当前节点的 pod 配置发生变化，就根据最新的配置执行响应的动作，保证运行的 pod 状态和期望的一致。这篇文章开始分析  kubelet 的源代码。"
categories: blog
tags: [kubernetes, kubelet, container, golang]
comments: true
share: true
---

## kubelet 简介

我在[之前的文章介绍过 kubelet 的功能](http://cizixs.com/2016/10/25/kubernetes-intro-kubelet)，简言之，kubelet 保证它所在节点的 pod 能够正常工作。它的核心工作是监听 apiserver，一旦发现当前节点的 pod 配置发生变化，就根据最新的配置执行响应的动作，保证运行的 pod 状态和期望的一致。

kubelet 除了这个最核心的功能之外，还有很多其他特性：

- 定时汇报当前节点的状态给 apiserver，以供调度的时候使用
- 镜像和容器的清理工作，保证节点上镜像不会占满磁盘空间，退出的容器不会占用太多资源
- 运行 HTTP Server，对外提供节点和 pod 信息，如果在 debug 模式下，还包括调试信息
- ……

从这篇文章开始，我们深入到 kubelet 的源码中，看它具体的实现原理。

**NOTE**：文章采用的 kubernetes 的版本是 `v1.5.0`，其他版本会有出入，请注意。因为 kubernetes 代码很繁杂，文章会适当删减，保证可读性。删除的内容包括但是不限于：

- 注释、TODO 等信息
- alpha 或者实验性特性代码
- 日志相关代码
- 参数验证、错误处理
- 和当前函数或者方法相关性很低的代码


## KubeletServer 配置对象

和其他 kubernetes 组件源代码一样，`kubelet` 的 `main` 函数入口放在 `cmd/` 文件夹下：

```
➜  kubernetes git:(v1.5.0) tree cmd/kubelet 
cmd/kubelet
├── app
│   ├── auth.go
│   ├── bootstrap.go
│   ├── bootstrap_test.go
│   ├── options
│   │   └── options.go
│   ├── plugins.go
│   ├── server.go
│   ├── server_linux.go
│   ├── server_test.go
│   └── server_unsupported.go
└── kubelet.go
```

`cmd` 是所有 kubernetes 组件的入口，主要负责二进制文件的命令行解析，和配置初始化工作，最终还是会调用 `pkg/` 下面各个组件的内容。对于 `kubelet` 来说，上面 `cmd/kubelet/kubelet.go` 就是 `main` 函数所在的文件，因为它的内容比较简单，所以就全部贴在这里了：

```go
func main() {
	runtime.GOMAXPROCS(runtime.NumCPU())
	// 注意这里，定义了 KubeletServer，主要用于一些参数的初始化和参数的定义
	s := options.NewKubeletServer()
	s.AddFlags(pflag.CommandLine)

    // 解析命令行参数
	flag.InitFlags()
	util.InitLogs()
	defer util.FlushLogs()

	verflag.PrintAndExitIfRequested()

    // 执行 `app.Run`，运行
	if err := app.Run(s, nil); err != nil {
		fmt.Fprintf(os.Stderr, "%v\n", err)
		os.Exit(1)
	}
}
```
这段代码主要分成三个部分，按照顺序运行的：

1. 创建一个 `KubeletServer` 对象，这个对象保存着 kubelet 运行需要的所有配置信息
2. 解析命令行，根据命令行的参数更新 `KubeletServer`
3. 根据 `KubeletServer` 的配置运行真正的 kubelet 程序

NOTE：第二部分是 `flag` 这个库自动完成的，因此我们只分析其他两个部分。

`options.NewKubeletServer()` 定义在 `app/options/options.go` 文件中，就是创建一个管理配置的结构体 `KubeletServer`，初始化一些配置信息。不要被 `KubeletServer` 这个名字迷惑，它只是一个包含了所有 kubelet 配置参数的结构体，并不是真正运行的 kubelet 实例。`KubeletServer` 对象结构是这样的：

```go
// KubeletServer 封装了运行 kubelet 需要的所有参数
type KubeletServer struct {

    // 主要的配置结构体，定义在 `apis/componentconfig/types.go` 文件中，包含了命令行所有可以配置的参数。
    // 因为这个字段是直接引用，所以用户可以通过 `KubeletServer` 直接访问它的字段
	componentconfig.KubeletConfiguration

    // kubeconfig 文件的路径，用于访问 apiserver，在后面的版本中这将成为访问 apiserver 的标准方式
	KubeConfig          flag.StringFlag
	BootstrapKubeconfig string

	// 如果设置为 true，那么错误的 kubeConfig 配置会直接导致 kubelet 退出
	RequireKubeConfig bool
	// 之前访问 apiserver 的方式，以后会被上面提到的 kubeConfig 配置取代
	AuthPath          flag.StringFlag // Deprecated -- use KubeConfig instead
	APIServerList     []string        // Deprecated -- use KubeConfig instead
	...
}
```

`KubeletServer` 有一个 `AddFlags` 的方法，它的唯一作用就是把命令行参数和它的字段一一对应起来。这样解析命令行参数的时候，就更新对应的字段。这里是所有命令行参数定义的地方，如果要查询某个版本提供了哪些命令行，我会阅读这部分内容。

后面所有的事情都是在 `app.Run()` 中做的，看名字也能猜出来，它会运行实际的 kubelet。这个方法定义在 `cmd/kubelet/app/server.go`：

```go
// 根据传进来的 kubeDeps 和 KubeletServer，运行 kubelet 的服务，这个方法会一直运行，正常情况下不会返回
func Run(s *options.KubeletServer, kubeDeps *kubelet.KubeletDeps) error {
	if err := run(s, kubeDeps); err != nil {
		return fmt.Errorf("failed to run Kubelet: %v", err)
	}
	return nil
}

func run(s *options.KubeletServer, kubeDeps *kubelet.KubeletDeps) (err error) {
	......
	if kubeDeps == nil {
		var kubeClient, eventClient *clientset.Clientset
		var cloud cloudprovider.Interface

		......

        // 创建出来两个 client：kubeClient 和 eventClient，用来和 apiserver 通信
		clientConfig, err := CreateAPIServerClientConfig(s)
		if err == nil {
			kubeClient, err = clientset.NewForConfig(clientConfig)
			if err != nil {
				glog.Warningf("New kubeClient from clientConfig error: %v", err)
			}
			// make a separate client for events
			eventClientConfig := *clientConfig
			eventClientConfig.QPS = float32(s.EventRecordQPS)
			eventClientConfig.Burst = int(s.EventBurst)
			eventClient, err = clientset.NewForConfig(&eventClientConfig)
		}
		......

        // 创建出来一个默认的 kubeDeps，里面包含了 dockerClient、Network Plugins 对象、Volume Plugins 对象
		kubeDeps, err = UnsecuredKubeletDeps(s)
		if err != nil {
			return err
		}
        // 把之前创建的对象赋给 kubeDeps
		kubeDeps.Cloud = cloud
		kubeDeps.KubeClient = kubeClient
		kubeDeps.EventClient = eventClient
	}

	......

    // 创建 cAdvisor 对象，负责收集容器的监控信息
	if kubeDeps.CAdvisorInterface == nil {
		kubeDeps.CAdvisorInterface, err = cadvisor.New(uint(s.CAdvisorPort), s.ContainerRuntime, s.RootDirectory)
		if err != nil {
			return err
		}
	}

    // 创建 ContainerManager 对象
	if kubeDeps.ContainerManager == nil {
	    ......
		kubeDeps.ContainerManager, err = cm.NewContainerManager(
			kubeDeps.Mounter,
			kubeDeps.CAdvisorInterface,
			cm.NodeConfig{
				RuntimeCgroupsName:    s.RuntimeCgroups,
				SystemCgroupsName:     s.SystemCgroups,
				KubeletCgroupsName:    s.KubeletCgroups,
				ContainerRuntime:      s.ContainerRuntime,
				CgroupsPerQOS:         s.ExperimentalCgroupsPerQOS,
				CgroupRoot:            s.CgroupRoot,
				CgroupDriver:          s.CgroupDriver,
				ProtectKernelDefaults: s.ProtectKernelDefaults,
				EnableCRI:             s.EnableCRI,
			},
			s.ExperimentalFailSwapOn)

		if err != nil {
			return err
		}
	}

	......

    // 运行 kubelet，这个函数会启动 goroutine 一直运行，是 kubelet 核心功能执行的地方
	if err := RunKubelet(&s.KubeletConfiguration, kubeDeps, s.RunOnce, standaloneMode); err != nil {
		return err
	}

    // 运行 healthz HTTP 服务
	if s.HealthzPort > 0 {
		healthz.DefaultHealthz()
		go wait.Until(func() {
			err := http.ListenAndServe(net.JoinHostPort(s.HealthzBindAddress, strconv.Itoa(int(s.HealthzPort))), nil)
			if err != nil {
				glog.Errorf("Starting health server failed: %v", err)
			}
		}, 5*time.Second, wait.NeverStop)
	}

    ......

	<-done
	return nil
}
```

这段代码的最后，是真正运行的东西。前面大部分的内容都是是在创建和初始化 `kubeDeps` 这个对象，它最终会传递到 `RunKubelet` 函数中。

`kubeDeps` 的名字听起来很奇怪，其实它内部保存了 kubelet 各个重要组件的对象，之所以要把它作为参数传递，是为了实现 dependency injection。简单地说，就是把 kubelet 依赖的组件对象作为参数传进来，这样可以控制 kubelet 的行为。比如在测试的时候，只要构建 fake 的组件实现，就能很轻松进行测试。`KubeDeps` 包含的组件很多，下面列出一些：

- CAdvisorInterface：提供 cAdvisor 接口功能的组件，用来获取监控信息
- DockerClient：docker 客户端，用来和 docker 交互
- KubeClient：apiserver 客户端，用来和 api server 通信
- Mounter：执行 mount 相关操作
- NetworkPlugins：网络插件，执行网络设置工作
- VolumePlugins：volume 插件，执行 volume 设置工作

可以看到，这些组件要么需要和第三方交互（kubeClient、DockerClient），要么有副作用（ `Mounter`、`NetworkPlugins`、`VolumePlugins`），在进行单元测试的时候一般都会编写对应的 Fake 对象，只要满足响应的接口，就能正常工作。

`run` 方法允许传进来的 `kubeDeps` 为空，这个时候它会自动生成默认的 `kubeDeps` 对象，这也就是我们上面代码的逻辑。运行 HTTP Server 的代码我们暂时略过，留作以后再讲，继续来看 `RunKubelet`，它的代码是这样的：

```go
func RunKubelet(kubeCfg *componentconfig.KubeletConfiguration, kubeDeps *kubelet.KubeletDeps, runOnce bool, standaloneMode bool) error {
    ......

    // 一些初始化的工作，配置 eventBroadcaster，这样就能发送事件了
	eventBroadcaster := record.NewBroadcaster()
	kubeDeps.Recorder = eventBroadcaster.NewRecorder(api.EventSource{Component: "kubelet", Host: string(nodeName)})
	eventBroadcaster.StartLogging(glog.V(3).Infof)
	if kubeDeps.EventClient != nil {
		glog.V(4).Infof("Sending events to api server.")
		eventBroadcaster.StartRecordingToSink(&unversionedcore.EventSinkImpl{Interface: kubeDeps.EventClient.Events("")})
	} else {
		glog.Warning("No api server defined - no events will be sent to API server.")
	}

	......

	privilegedSources := capabilities.PrivilegedSources{
		HostNetworkSources: hostNetworkSources,
		HostPIDSources:     hostPIDSources,
		HostIPCSources:     hostIPCSources,
	}
	capabilities.Setup(kubeCfg.AllowPrivileged, privilegedSources, 0)

    ......

    // 使用 Builder 创建 mainKubelet，默认使用 `CreateAndInitKubelet`
	builder := kubeDeps.Builder
	if builder == nil {
		builder = CreateAndInitKubelet
	}
	if kubeDeps.OSInterface == nil {
		kubeDeps.OSInterface = kubecontainer.RealOS{}
	}
	k, err := builder(kubeCfg, kubeDeps, standaloneMode)
    
    ......
	// 运行 kubelet
	if runOnce {
		if _, err := k.RunOnce(podCfg.Updates()); err != nil {
			return fmt.Errorf("runonce failed: %v", err)
		}
		glog.Infof("Started kubelet %s as runonce", version.Get().String())
	} else {
		err := startKubelet(k, podCfg, kubeCfg, kubeDeps)
		if err != nil {
			return err
		}
		glog.Infof("Started kubelet %s", version.Get().String())
	}
	return nil
}
```

`RunKubelet()` 和 `runKubelet()` 完成了真正意义上的 `kubelet` 对象的创建和运行。`RunKubelet` 用来创建 kubelet、验证参数、以及做一些初始化（事件处理和配置 capabilities 等），`runKubelet` 负责运行。

`RunKubelet` 的内容可以分成三个部分：

1. 初始化各个对象，比如 eventBroadcaster，这样就能给 apiserver 发送 kubelet 的事件
2. 通过 builder 创建出来 `Kubelet` 
3. 根据运行模式，运行 `Kubelet`

创建工作是在 `k, err := builder(kubeCfg, kubeDeps, standaloneMode)` 这句完成的，默认的 `builder` 是 `CreateAndInitKubelet`：

```go
func CreateAndInitKubelet(kubeCfg *componentconfig.KubeletConfiguration, kubeDeps *kubelet.KubeletDeps, standaloneMode bool) (k kubelet.KubeletBootstrap, err error) {

    // 调用 pkg/kubelet/kubelet.go 文件创建 MainKubelet 的代码
	k, err = kubelet.NewMainKubelet(kubeCfg, kubeDeps, standaloneMode)
	if err != nil {
		return nil, err
	}

	k.BirthCry()

    // 启动 GC
	k.StartGarbageCollection()

	return k, nil
}
```

`BirthCry()` 只是发送一个事件，宣告 kubelet 已经成功启动；`StartGarbageCollection()` 正如名字所示，启动 kubelet GC 流程，我们后面会详细解释它的内部实现。接下来，我们先看一下 `NewMainKubelet()` 的代码，毕竟它是创建出 kubelet 的地方。

## kubelet 的创建

`MainKubelet` 函数定义在 `pkg/kubelet/kubelet.go#NewMainKubelet`，我们终于从 `cmd/kubelet/` 分析到 `pkg/kubelet/` 了。

```go
func NewMainKubelet(kubeCfg *componentconfig.KubeletConfiguration, kubeDeps *KubeletDeps, standaloneMode bool) (*Kubelet, error) {
	......

    // PodConfig 非常重要，它是 pod 信息的来源，kubelet 支持文件、URL 和 apiserver 三种渠道，PodConfig 将它们汇聚到一起，通过管道来传递
	if kubeDeps.PodConfig == nil {
		kubeDeps.PodConfig, err = makePodSourceConfig(kubeCfg, kubeDeps, nodeName)
	}
    ......

    // exec 处理函数，进入到容器中执行命令的方式。之前使用的是 nsenter 命令行的方式，后来 docker 提供了 `docker exec` 命令，默认是后者
	var dockerExecHandler dockertools.ExecHandler
	switch kubeCfg.DockerExecHandlerName {
	case "native":
		dockerExecHandler = &dockertools.NativeExecHandler{}
	case "nsenter":
		dockerExecHandler = &dockertools.NsenterExecHandler{}
	default:
		glog.Warningf("Unknown Docker exec handler %q; defaulting to native", kubeCfg.DockerExecHandlerName)
		dockerExecHandler = &dockertools.NativeExecHandler{}
	}

    // 使用 reflector 把 ListWatch 得到的服务信息实时同步到 serviceStore 对象中
	serviceStore := cache.NewIndexer(cache.MetaNamespaceKeyFunc, cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc})
	if kubeClient != nil {
		serviceLW := cache.NewListWatchFromClient(kubeClient.Core().RESTClient(), "services", api.NamespaceAll, fields.Everything())
		cache.NewReflector(serviceLW, &api.Service{}, serviceStore, 0).Run()
	}
	serviceLister := &cache.StoreToServiceLister{Indexer: serviceStore}

    // 使用 reflector 把 ListWatch 得到的节点信息实时同步到  nodeStore 对象中
	nodeStore := cache.NewStore(cache.MetaNamespaceKeyFunc)
	if kubeClient != nil {
		fieldSelector := fields.Set{api.ObjectNameField: string(nodeName)}.AsSelector()
		nodeLW := cache.NewListWatchFromClient(kubeClient.Core().RESTClient(), "nodes", api.NamespaceAll, fieldSelector)
		cache.NewReflector(nodeLW, &api.Node{}, nodeStore, 0).Run()
	}
	nodeLister := &cache.StoreToNodeLister{Store: nodeStore}
	nodeInfo := &predicates.CachedNodeInfo{StoreToNodeLister: nodeLister}

    ......

    // 根据配置信息和各种对象创建 Kubelet 实例
	klet := &Kubelet{
		hostname:                       hostname,
		nodeName:                       nodeName,
		dockerClient:                   kubeDeps.DockerClient,
		kubeClient:                     kubeClient,
		......
		clusterDomain:                  kubeCfg.ClusterDomain,
		clusterDNS:                     net.ParseIP(kubeCfg.ClusterDNS),
		serviceLister:                  serviceLister,
		nodeLister:                     nodeLister,
		nodeInfo:                       nodeInfo,
		masterServiceNamespace:         kubeCfg.MasterServiceNamespace,
		streamingConnectionIdleTimeout: kubeCfg.StreamingConnectionIdleTimeout.Duration,
		recorder:                       kubeDeps.Recorder,
		cadvisor:                       kubeDeps.CAdvisorInterface,
		diskSpaceManager:               diskSpaceManager,
        ......
	}

	......

    // 网络插件的初始化工作
	if plug, err := network.InitNetworkPlugin(kubeDeps.NetworkPlugins, kubeCfg.NetworkPluginName, &criNetworkHost{&networkHost{klet}}, klet.hairpinMode, klet.nonMasqueradeCIDR, int(kubeCfg.NetworkPluginMTU)); err != nil {
		return nil, err
	} else {
		klet.networkPlugin = plug
	}
    
    // 从 cAdvisor 获取当前机器的信息
	machineInfo, err := klet.GetCachedMachineInfo()
	......

	procFs := procfs.NewProcFS()
	imageBackOff := flowcontrol.NewBackOff(backOffPeriod, MaxContainerBackOff)

	klet.livenessManager = proberesults.NewManager()
    
    // podManager 负责管理当前节点上的 pod 信息，它保存了所有 pod 的内容，包括 static pod。
    // kubelet 从本地文件、网络地址和 apiserver 三个地方获取 pod 的内容，
	klet.podCache = kubecontainer.NewCache()
	klet.podManager = kubepod.NewBasicPodManager(kubepod.NewBasicMirrorClient(klet.kubeClient))
    ......

    // 创建 runtime 对象，以后会改用 CRI 接口和 runtime 交互，目前使用 DockerManager
	if kubeCfg.EnableCRI {
        ......
	} else {
		switch kubeCfg.ContainerRuntime {
		case "docker":
			runtime := dockertools.NewDockerManager(
				kubeDeps.DockerClient,
				kubecontainer.FilterEventRecorder(kubeDeps.Recorder),
				klet.livenessManager,
				containerRefManager,
				klet.podManager,
				machineInfo,
				kubeCfg.PodInfraContainerImage,
				float32(kubeCfg.RegistryPullQPS),
				int(kubeCfg.RegistryBurst),
				ContainerLogsDir,
				kubeDeps.OSInterface,
				klet.networkPlugin,
				klet,
				klet.httpClient,
				dockerExecHandler,
				kubeDeps.OOMAdjuster,
				procFs,
				klet.cpuCFSQuota,
				imageBackOff,
				kubeCfg.SerializeImagePulls,
				kubeCfg.EnableCustomMetrics,
				klet.hairpinMode == componentconfig.HairpinVeth && kubeCfg.NetworkPluginName != "kubenet",
				kubeCfg.SeccompProfileRoot,
				kubeDeps.ContainerRuntimeOptions...,
			)
			klet.containerRuntime = runtime
			klet.runner = kubecontainer.DirectStreamingRunner(runtime)
		case "rkt":
			......
		default:
			return nil, fmt.Errorf("unsupported container runtime %q specified", kubeCfg.ContainerRuntime)
		}
	}

    ......
    
	klet.pleg = pleg.NewGenericPLEG(klet.containerRuntime, plegChannelCapacity, plegRelistPeriod, klet.podCache, clock.RealClock{})
	klet.runtimeState = newRuntimeState(maxWaitForContainerRuntime)
	klet.updatePodCIDR(kubeCfg.PodCIDR)

	// 创建 containerGC 对象，进行周期性的容器清理工作
	containerGC, err := kubecontainer.NewContainerGC(klet.containerRuntime, containerGCPolicy)
	if err != nil {
		return nil, err
	}
	klet.containerGC = containerGC
	klet.containerDeletor = newPodContainerDeletor(klet.containerRuntime, integer.IntMax(containerGCPolicy.MaxPerPodContainer, minDeadContainerInPod))

	// 创建 imageManager 对象，管理镜像
	imageManager, err := images.NewImageGCManager(klet.containerRuntime, kubeDeps.CAdvisorInterface, kubeDeps.Recorder, nodeRef, imageGCPolicy)
	if err != nil {
		return nil, fmt.Errorf("failed to initialize image manager: %v", err)
	}
	klet.imageManager = imageManager

    // statusManager 实时检测节点上 pod 的状态，并更新到 apiserver 对应的 pod 
	klet.statusManager = status.NewManager(kubeClient, klet.podManager)

    // probeManager 检测 pod 的状态，并通过 statusManager 进行更新
	klet.probeManager = prober.NewManager(
		klet.statusManager,
		klet.livenessManager,
		klet.runner,
		containerRefManager,
		kubeDeps.Recorder)

    // volumeManager 管理节点上 volume
    
	klet.volumePluginMgr, err =
		NewInitializedVolumePluginMgr(klet, kubeDeps.VolumePlugins)
	if err != nil {
		return nil, err
	}
	......
	// setup volumeManager
	klet.volumeManager, err = volumemanager.NewVolumeManager(
		kubeCfg.EnableControllerAttachDetach,
		nodeName,
		klet.podManager,
		klet.kubeClient,
		klet.volumePluginMgr,
		klet.containerRuntime,
		kubeDeps.Mounter,
		klet.getPodsDir(),
		kubeDeps.Recorder,
		kubeCfg.ExperimentalCheckNodeCapabilitiesBeforeMount)

    // 保存了节点上正在运行的 pod 信息
	runtimeCache, err := kubecontainer.NewRuntimeCache(klet.containerRuntime)
	if err != nil {
		return nil, err
	}
	klet.runtimeCache = runtimeCache
	klet.reasonCache = NewReasonCache()
	klet.workQueue = queue.NewBasicWorkQueue(klet.clock)
	
	// podWorkers 是具体的执行者
	klet.podWorkers = newPodWorkers(klet.syncPod, kubeDeps.Recorder, klet.workQueue, klet.resyncInterval, backOffPeriod, klet.podCache)

	......
	klet.kubeletConfiguration = *kubeCfg
	return klet, nil
}
```

`NewMainKubelet` 正如名字所示，主要的工作就是创建 `Kubelet` 这个对象，它包含了 kubelet 运行需要的所有对象，上面的代码就是各种对象的初始化和赋值的过程，这里只介绍几个非常重要的对象来说：

- podConfig：这个对象里面会从文件、网络和 apiserver 三个来源中汇聚节点要运行的 pod 信息，并通过管道发送出来，读取这个管道就能获取实时的 pod 最新配置
- ServiceLister：能够读取 kubernetes 中服务信息
- nodeLister：能够读取 apiserver 中节点的信息
- diskSpaceManager：返回容器存储空间的信息
- podManager：缓存了 pod 的信息，是所有需要该信息都会去访问的地方
- runtime：容器运行时，对容器引擎（docker 或者 rkt）的一层封装，负责调用容器引擎接口管理容器的状态，比如启动、暂停、杀死容器等
- probeManager：如果 pod 配置了状态监测，那么 probeManager 会定时检查 pod 是否正常工作，并通过 statusManager 向 apiserver 更新 pod 的状态
- volumeManager：负责容器需要的 volume 管理。检测某个 volume 是否已经 mount、获取 pod 使用的 volume 等
- podWorkers：具体的执行者，每次有 pod 需要更新的时候都会发送给它

这里并不一一展开所有对象的实现和具体功能，以后的文章会对其中一些继续分析。

## kubelet 的运行

接着回到 kubelet 的分析，通过 `NewMainKubelet` 创建完 `Kubelet` 对象，下一步就是看看如果运行它。在 `RunKubelet` 内部，我们看到它最终会调用 `startKublet` 函数：

```go
func startKubelet(k kubelet.KubeletBootstrap, podCfg *config.PodConfig, kubeCfg *componentconfig.KubeletConfiguration, kubeDeps *kubelet.KubeletDeps) error {
	// 启动 kubelet
	go wait.Until(func() { k.Run(podCfg.Updates()) }, 0, wait.NeverStop)

	// 启动 kubelet server
	if kubeCfg.EnableServer {
		go wait.Until(func() {
			k.ListenAndServe(net.ParseIP(kubeCfg.Address), uint(kubeCfg.Port), kubeDeps.TLSOptions, kubeDeps.Auth, kubeCfg.EnableDebuggingHandlers)
		}, 0, wait.NeverStop)
	}
	if kubeCfg.ReadOnlyPort > 0 {
		go wait.Until(func() {
			k.ListenAndServeReadOnly(net.ParseIP(kubeCfg.Address), uint(kubeCfg.ReadOnlyPort))
		}, 0, wait.NeverStop)
	}

	return nil
}
```

运行 kubelet 主要启动两个功能，`k.Run()` 来进入主循环，`k.ListenAndServe()` 启动 kubelet 的 API 服务，后者并不是这篇文章的重点，我们来看看前者，它的执行入口是 `k.Run(podCfg.Updates())`，`podCfg.Updates()` 我们前面已经说过，它是一个管道，会实时地发送过来 pod 最新的配置信息，至于是怎么实现的，我们以后再说，这里知道它的作用就行。`Run` 方法的代码如下：

```go
/ Run starts the kubelet reacting to config updates
func (kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate) {
    .....

	// Start volume manager
	go kl.volumeManager.Run(kl.sourcesReady, wait.NeverStop)

    // 定时向 apiserver 更新 node 信息，用作调度时的重要参考
	if kl.kubeClient != nil {
		// Start syncing node status immediately, this may set up things the runtime needs to run.
		go wait.Until(kl.syncNodeStatus, kl.nodeStatusUpdateFrequency, wait.NeverStop)
	}
	go wait.Until(kl.syncNetworkStatus, 30*time.Second, wait.NeverStop)
	go wait.Until(kl.updateRuntimeUp, 5*time.Second, wait.NeverStop)

	// Start loop to sync iptables util rules
	if kl.makeIPTablesUtilChains {
		go wait.Until(kl.syncNetworkUtil, 1*time.Minute, wait.NeverStop)
	}

	// 删除 podWorker 没有正常处理的 pod 
	go wait.Until(kl.podKiller, 1*time.Second, wait.NeverStop)

	// Start component sync loops.
	// 管理 pod 和容器的状态
	kl.statusManager.Start()
	
	// readiness 和 liveness 管理
	kl.probeManager.Start()

	// pleg  的全称是 Pod Lifecycle Event Generator
	kl.pleg.Start()
	
	// 宫殿的入口
	kl.syncLoop(updates, kl)
}
```

基本上就是 `kubelet` 各种组件的启动，每个组件都是以 goroutine 运行的，这里不做赘述。最后一句 `kl.syncLoop(updates, kl)` 是处理所有 pod 更新的主循环，获取 pod 的变化（新建、修改和删除），调用对应的处理函数保证节点上的容器符合 pod 的配置。

如果 kubelet 是做宫殿，那么 `syncLoop` 就是这座宫殿的入口。我们从很远的地方出发，穿过丛林、越过高山，一路上到处问路，经过一座座的城镇，终于抵达都城。穿行过眼花缭乱的商铺、士兵、民宅，站在金碧辉煌巍峨高大的宫殿门口，心中一定激情澎湃。

在下一篇文章，我们就要打开宫殿大门，一探其中究竟。

## 参考资料

- [k8s kubelet源码剖析](http://www.webpaas.com/index.php/archives/120/)
- [kubelet源码解析(1)](http://licyhust.com/%E5%AE%B9%E5%99%A8%E6%8A%80%E6%9C%AF/2016/10/20/kubelet-1/)
- [KUBERNETES NODE COMPONENTS – KUBELET](http://www.sel.zju.edu.cn/?p=595)
- [kubernetes源码分析 -- kubelet组件](http://blog.csdn.net/zhaoguoguang/article/details/51225553)