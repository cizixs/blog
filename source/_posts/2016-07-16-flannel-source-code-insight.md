---
layout: post
title: "flannel 源码分析"
excerpt: "flannel 是 Coreos 开发的解决容器跨主机网络的项目，这篇文章试着分析它的源代码，希望能加深对它的理解和掌握。"
categories: blog
tags: [flannel, container, network]
comments: true
share: true
---

## 简介

`flannel` 是为了解决容器的跨主机网络问题而出现的项目，可以提供多种类型的网络模型：

- 普通网络：udp、vxlan、hostgw
- 平台网络：gce、aws

使用之前，需要在 etcd 中写入要管理网络的配置，比如 ：

    {
        "Network": "10.0.0.0/8",
        "SubnetLen": 20,
        "SubnetMin": "10.10.0.0",
        "SubnetMax": "10.99.0.0",
        "Backend": {
            "Type": "udp",
            "Port": 7890
        }
    }

这里有三个概念要了解：

- 网络（Network）：整个集群中分配给 flannel 要管理的网络地址范围
- 子网（Subnet）：flannel 所在的每台主机都会管理 network 中一个子网，子网的掩码和范围是可配置的
- 后端（Backend）：使用什么样的后端网络模型，比如默认的 udp，还是 vxlan 等

flannel 在 `etcd` 中保存的数据默认保存在 `/coreos.com/network`，其中

- `/coreos.com/network/config` 保存着上面网络配置数据
- `/coreos.com/network/subnets` 是一个目录，保存的是每个主机分配的子网，比如

      {
      	"createdIndex": 5,
      	"expiration": "2016-07-15T02:44:46.411907661Z",
      	"key": "/coreos.com/network/subnets/10.16.240.0-20",
      	"modifiedIndex": 5,
      	"ttl": 66125,
      	"value": "{\"PublicIP\":\"192.168.8.101\",\"BackendType\":\"host-gw\"}"
      }

`key` 里面保存着子网的网段信息，`value` 保存着租约信息（lease），`expiration` 是过期时间。

更多关于 flannel 的配置和使用，请参考我之前写的 [flannel 网络模型](http://cizixs.com/2016/06/15/flannel-overlay-network) 这篇文章，或者阅读官方文档。

## 代码结构

    ├── Documentation/    # 文档
    ├── backend/          # 后端实现，目前支持 udp、vxlan、host-gw 等
    ├── dist/
    ├── logos/
    ├── network/          # 最上层的网络管理逻辑
    ├── pkg/              # 抽象出来的功能库，目前只有 `ip`
    ├── remote/
    ├── subnet/           # 子网管理功能
    ├── vendor/
    ├── version/
    ├── CONTRIBUTING.md
    ├── Dockerfile
    ├── LICENSE
    ├── MAINTAINERS
    ├── NOTICE
    ├── README.md
    ├── build*
    ├── cover*
    ├── main.go           # 可执行文件的入口
    ├── packet-01.png
    └── test*


## 代码分析

flannel 最近有两个新的使用模式：server-client 和 多网络，因为目前还不是稳定阶段，这篇文章会跳过相关的内容，只考虑每个容器/主机只有一个网络的情况。

### **network**

`flannel/network` 文件夹有三个文件：

- `ipmasq.go`：负责 iptables masquerade 功能的配置和删除
- `manager.go`：最上层的封装和最终的接口 API
- `network.go`：网络的主要逻辑，也是 loop 循环逻辑所在的地方

`main.go` 是 flannel 代码的入口，这个文件只做了一件事情：创建 NetworkManager，并执行它的 `Run` 函数。精简后的代码逻辑是这样的：

    # 创建 subnetManager，作为参数城传递给 NetworkManager
    sm, err := newSubnetManager()
    nm, err := network.NewNetworkManager(ctx, sm)

    # 运行 NetworkManager 的 run 函数，并等待它结束
    wg := sync.WaitGroup{}
    wg.Add(1)
    go func() {
      	nm.Run(ctx)
    	wg.Done()
    }()

    <-sigs
    // unregister to get default OS nuke behaviour in case we don't exit cleanly
    signal.Stop(sigs)

    log.Info("Exiting...")
    cancel()

    wg.Wait()

这段代码虽然简单，但是也带来了更多的疑惑：

- `NetworkManager` 是什么？有什么作用？
- `SubnetManager` 又是什么？为什么要作为参数传给 `NetworkManager`？
- `NetworkManager.Run(ctx)` 函数主要做了哪些事情呢？

下面所有的内容就是为了回答这些问题，我们先看一下 `NetworkManager` ，它定义在 `flannel/network/manager.go` 文件中，结构是这样的：

    type Manager struct {
    	ctx             context.Context        // 请求的上下文
    	sm              subnet.Manager         // 子网管理，每台 flannel 节点都是一个子网
    	bm              backend.Manager        // 后端管理，比如常用的 udp、xvlan、host-gw
    	allowedNetworks map[string]bool        // 多网络模式会用到
    	mux             sync.Mutex           
    	networks        map[string]*Network    // 多网络模式下，要管理的网络列表
    	watch           bool
    	ipMasq          bool                   
    	extIface        *backend.ExternalInterface  // 和外部节点通信的网络接口，比如 eth0
    }

而上面的 `NewNetworkManager` 就是为了创建这样一个对象，创建的时候只传入了 `ctx` 和 `sm` 两个值，其他的值会自动创建，其中比较重要的是 `backend.Manager`，负责管理后端支持的插件。

`NetworkManager` 是外层的抽象，根据 etcd 中的网络配置主机的网络信息。我们看一下 `Manager.Run` 就知道啦：

    func (m *Manager) Run(ctx context.Context) {
    	wg := sync.WaitGroup{}
    	# 添加要管理的网络
    	m.networks[""] = NewNetwork(ctx, m.sm, m.bm, "", m.ipMasq)

    	// 对于每一个网络，调用 `runNetwork`
    	m.forEachNetwork(func(n *Network) {
    		wg.Add(1)
    		go func(n *Network) {
    			m.runNetwork(n)
    			wg.Done()
    		}(n)
    	})

    	wg.Wait()
    	m.bm.Wait()
    }

这里的内容也不多，无非是添加要管理的网络，并调用 `runNetwork` 函数。`runNetwork` 函数会完成实际的工作：

    func (m *Manager) runNetwork(n *Network) {
    	n.Run(m.extIface, func(bn backend.Network) {
    		log.Infof("Lease acquired: %v", bn.Lease().Subnet)

    		if err := writeSubnetFile(opts.subnetFile, n.Config.Network, m.ipMasq, bn); err != nil {
    			log.Warningf("%v failed to write subnet file: %s", n.Name, err)
    			return
    		}
    		daemon.SdNotify("READY=1")
    	})

    	m.delNetwork(n)
    }

调用 `network.Run`，第一个参数是网络接口，第二个参数是一个初始化的函数（这个函数的工作就是根据网络内容，把配置写到本地的文件）。

还记得这个 `Network` 变量是怎么生成的吗？看一下上面 `m.networks[""] = NewNetwork(ctx, m.sm, m.bm, "", m.ipMasq)` 这句，这个对象的代码在 `flannel/network/network.go` 这个文件中。`Network` 的结构是这样的：

    type Network struct {
    	Name   string
    	Config *subnet.Config

    	ctx        context.Context
    	cancelFunc context.CancelFunc
    	sm         subnet.Manager
    	bm         backend.Manager
    	ipMasq     bool
    	bn         backend.Network
    }

`network.Run` 就是循环调用 `runOnce` 的逻辑，那么我们看看 `runOnce` 这个函数，这部分的代码比较长，也是 `flannel/network` 文件夹中最核心的内容。我在关键部分已经添加了中文注释：

    func (n *Network) runOnce(extIface *backend.ExternalInterface, inited func(bn backend.Network)) error {
        # retryInit 做一下初始化的工作，主要是从 etcd 中获取网络的配置，根据里面的数据
        # 获取对应的 backend、从管理的网络中分配一个子网、把分配的子网保存到 backend 的数据结构中。
        # 分配网络的工作 `subnet/` 文件下的代码实现的，后面会详细说明
    	if err := n.retryInit(); err != nil {
    		return errCanceled
    	}

        # 运行传进来的初始化工作函数，就是上面提到的在本地创建 subnet.env 文件的函数
    	inited(n.bn)

    	ctx, interruptFunc := context.WithCancel(n.ctx)

    	wg := sync.WaitGroup{}

    	# 调用 backend.network 的 Run 函数，在后台运行；
    	# 每个 backend 的运行逻辑不同，但思路都是：监听 etcd 的变化，并根据变化后的内容执行命令
    	wg.Add(1)
    	go func() {
    		n.bn.Run(ctx)
    		wg.Done()
    	}()

    	evts := make(chan subnet.Event)

        # 监听子网所在 etcd 的变化，如果
    	wg.Add(1)
    	go func() {
    		subnet.WatchLease(ctx, n.sm, n.Name, n.bn.Lease().Subnet, evts)
    		wg.Done()
    	}()

        # 在运行结束的时候，拆除设置的 ipmasq
    	defer func() {
    		if n.ipMasq {
    			if err := teardownIPMasq(n.Config.Network); err != nil {
    				log.Errorf("Failed to tear down IP Masquerade for network %v: %v", n.Name, err)
    			}
    		}
    	}()

    	defer wg.Wait()


    	dur := n.bn.Lease().Expiration.Sub(time.Now()) - renewMargin
    	for {
    		select {
    		# 超时，要进行续租
    		case <-time.After(dur):
    			err := n.sm.RenewLease(n.ctx, n.Name, n.bn.Lease())
    			if err != nil {
    				log.Error("Error renewing lease (trying again in 1 min): ", err)
    				dur = time.Minute
    				continue
    			}

    			log.Info("Lease renewed, new expiration: ", n.bn.Lease().Expiration)
    			dur = n.bn.Lease().Expiration.Sub(time.Now()) - renewMargin

    		# 收到监听的事件，进行处理。目前只有两种事件：子网添加，和子网删除
    		case e := <-evts:
    			switch e.Type {
    			case subnet.EventAdded:
    				n.bn.Lease().Expiration = e.Lease.Expiration
    				dur = n.bn.Lease().Expiration.Sub(time.Now()) - renewMargin

    			case subnet.EventRemoved:
    				log.Warning("Lease has been revoked")
    				interruptFunc()
    				return errInterrupted
    			}

    		case <-n.ctx.Done():
    			return errCanceled
    		}
    	}
    }

总结起来也比较简单：

- 读取 etcd 中的值，根据获得的值做一些初始化的工作
- 调用 backend 的 Run 函数，让 backend 在后台运行
- 监听子网在 etcd 中的变化，执行对应的操作

### subnet

在前一部分，network 最后初始化的步骤（`n.retryInit()`）中，可能很多人和我一样，比较关心每个主机分配的子网是怎么决定的，这个秘密在 `subnet/local_manager.go:allocateSubnet()` 函数里：

    var bag []ip.IP4
    sn := ip.IP4Net{IP: config.SubnetMin, PrefixLen: config.SubnetLen}

    # 没错，我也看到这个位置标识符了，能看到它为什么存在吗？
    # 这个东西其实很容器被处理掉，能想到好的方法吗？
    OuterLoop:
        # 从低到高，依次列举所有可以分配的子网(上限是 100)，找到那些和已分配的网络没有冲突的，放到
        # 数组中备选
    	for ; sn.IP <= config.SubnetMax && len(bag) < 100; sn = sn.Next() {
    		for _, l := range leases {
    			if sn.Overlaps(l.Subnet) {
    				continue OuterLoop
    			}
    		}
    		bag = append(bag, sn.IP)
    	}

    	# 从可用的子网中随机选择一个返回，如果没有可用的，报错。
    	if len(bag) == 0 {
    		return ip.IP4Net{}, errors.New("out of subnets")
    	} else {
    		i := randInt(0, len(bag))
    		return ip.IP4Net{IP: bag[i], PrefixLen: config.SubnetLen}, nil
    	}

`subnet` 目录所有子网管理的文件，负责子网的创建、更新、添加、删除、监听等，主要和 `etcd` 打交道：

    subnet
    ├── clock.go
    ├── config.go         # 把 etcd 中获取的结果转换为程序的数据结构
    ├── config_test.go
    ├── local_manager.go  # subnet manager， 获取子网的分配、续约、监听租约等
    ├── mock_etcd.go      # etcd mock 对象，用于测试和本地开发
    ├── mock_etcd_test.go
    ├── mock_registry.go  # registry mock 对象
    ├── mock_subnet.go    # subnet mock 对象，实现了子网的管理功能
    ├── rand.go           # 帮助函数，实现了一个随机函数，用于申请子网时的随机过程
    ├── registry.go       # etcd 的封装对象，实现网络数据在 etcd 中的 CRUD 等操作
    ├── registry_test.go
    ├── subnet.go         # 子网的隔离功能
    ├── subnet_test.go
    └── watch.go          # 监听 etcd 的数据结构功能，包括网络监听、租期监听等

### backend

`backend` 文件夹汇集了不同的后端实现， `flannel` 在运行的时候回根据 etcd 中的配置来选择对应的后端，并执行它的 `Run` 函数。backend 的接口比较简单，只有两个函数：

    type Backend interface {
    	// Called first to start the necessary event loops and such
    	Run(ctx context.Context)
    	// Called when the backend should create or begin managing a new network
    	RegisterNetwork(ctx context.Context, network string, config *subnet.Config) (Network, error)
    }

- `Run` 函数是 backend 初始化的时候被调用的，用来启动 event loop，接受关闭信号，以便做一些清理工作
- `RegisterNetwork` 函数在需要后端要管理某个网络的时候被调用，目标就是注册要管理的网络信息到自己的数据结构中，以便后面使用

具体的工作在每个 backend 的 network 中执行，`backend/network` 的接口如下：

    type Network interface {
    	Lease() *subnet.Lease
    	MTU() int
    	Run(ctx context.Context)
    }

- `Lease` 返回后端管理的子网租期信息
- `MTU` 返回后端锁管理网络的 MTU（Maximum Transmission Unit）
- `Run` 执行后端的核心逻辑

我们以 `hostgw` 后端为例（它比较简单，因此容易理解）重点看一下 `Network.Run` 函数。`hostgw` 模式下，每台主机上的 flannel 只负责路由表的维护就行了，当发现 etcd 中有节点信息变化的时候就随时更新自己主机的路由表项。我们来看一下它的代码：

    func (n *network) Run(ctx context.Context) {
    	wg := sync.WaitGroup{}

        # 创建一个 Event 的 channel，用来存储监听过程中出现的事件
    	log.Info("Watching for new subnet leases")
    	evts := make(chan []subnet.Event)
    	wg.Add(1)

    	# 调用 goroutine 在后台监听子网租期的变化，把结果放到 evts 里面传回来
    	go func() {
    		subnet.WatchLeases(ctx, n.sm, n.name, n.lease, evts)
    		wg.Done()
    	}()

        # 另外一个后台程序，用来保证主机上的路由表和 etcd 中数据保持一致
        # 注意：这个程序只负责添加，不负责删除。如果管理的路由表项被手动删除了，或者重启失效，会自动添加上去
    	n.rl = make([]netlink.Route, 0, 10)
    	wg.Add(1)
    	go func() {
    		n.routeCheck(ctx)
    		wg.Done()
    	}()

    	defer wg.Wait()

    	# event loop：处理监听到的事件，程序退出的时候返回
    	for {
    		select {
    		case evtBatch := <-evts:
    			n.handleSubnetEvents(evtBatch)

    		case <-ctx.Done():
    			return
    		}
    	}
    }

做的事情一句话就能总结出来：监听 etcd 中的子网信息，有事件发生的时候调用对应的处理函数。处理函数只有负责两种时间：子网被添加和子网被删除。

    func (n *network) handleSubnetEvents(batch []subnet.Event) {
    	for _, evt := range batch {
    		switch evt.Type {
    		# 添加子网事件发生时的处理步骤：
    		# 检查参数是否正常，根据参数构建路由表项，把路由表项添加到主机，把路由表项添加到自己的数据结构中
    		case subnet.EventAdded:
    			log.Infof("Subnet added: %v via %v", evt.Lease.Subnet, evt.Lease.Attrs.PublicIP)

    			if evt.Lease.Attrs.BackendType != "host-gw" {
    				log.Warningf("Ignoring non-host-gw subnet: type=%v", evt.Lease.Attrs.BackendType)
    				continue
    			}

    			route := netlink.Route{
    				Dst:       evt.Lease.Subnet.ToIPNet(),
    				Gw:        evt.Lease.Attrs.PublicIP.ToIP(),
    				LinkIndex: n.linkIndex,
    			}
    			if err := netlink.RouteAdd(&route); err != nil {
    				log.Errorf("Error adding route to %v via %v: %v", evt.Lease.Subnet, evt.Lease.Attrs.PublicIP, err)
    				continue
    			}
    			n.addToRouteList(route)

    		# 删除子网事件发生时的处理步骤：
    		# 检查参数是否正常，根据参数构建路由表项，把路由表项从主机删除，把路由表项从管理的数据结构中删除
    		case subnet.EventRemoved:
    			log.Info("Subnet removed: ", evt.Lease.Subnet)

    			if evt.Lease.Attrs.BackendType != "host-gw" {
    				log.Warningf("Ignoring non-host-gw subnet: type=%v", evt.Lease.Attrs.BackendType)
    				continue
    			}

    			route := netlink.Route{
    				Dst:       evt.Lease.Subnet.ToIPNet(),
    				Gw:        evt.Lease.Attrs.PublicIP.ToIP(),
    				LinkIndex: n.linkIndex,
    			}
    			if err := netlink.RouteDel(&route); err != nil {
    				log.Errorf("Error deleting route to %v: %v", evt.Lease.Subnet, err)
    				continue
    			}
    			n.removeFromRouteList(route)

    		default:
    			log.Error("Internal error: unknown event type: ", int(evt.Type))
    		}
    	}
    }

需要注意的是：为了容错（fault tolerance），添加和删除路由出错的时候只是记一条 log，然后就跳过。在极端的情况下，会出现本地的路由表项和 etcd 中数据不一致的情况！

其他后端虽然实际上做的事情不同，但是思路都是一样的。这里就不一一说明了，有兴趣的可以自己去读源码。

## 总结

最后，我们来总结讲的内容，对最开始提出的问题给出答案。`flannel` 代码重要的有三个概念：network、subnet 和 backend。

- network 负责网络的管理（以后的方向是多网络模型），根据每个网络的配置调用 subnet；
- subnet 负责和 etcd 交互，把 etcd 中的信息转换为 flannel 的子网数据结构，并对 etcd 进行子网和网络的监听；
- backend 接受 subnet 的监听事件，并做出对应的处理。

## 参考文档

- [flannel github 官方源码](https://github.com/coreos/flannel)
