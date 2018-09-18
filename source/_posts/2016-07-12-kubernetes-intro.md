---
layout: post
title: "kubernetes 简介：kubernetes 架构介绍"
excerpt: "kubernetes 是谷歌开源的一套容器管理平台，这篇文章介绍它的整体架构，包括概念和组件。"
categories: blog
tags: [kubernetes, container, etcd, architecture]
comments: true
share: true
---

## 什么是 kubernetes？

kubernetes （经常被缩写成 k8s）是 google 开源的一套自动化容器管理平台，前身是 Borg，用于容器的部署、自动化调度和集群管理。目前 kubernetes 有以下的特性：

- 容器的自动化部署
- 自动化扩展或者缩容
- 自动化应用/服务升级
- 容器成组，对外提供服务，支持负载均衡
- 服务的健康检查，自动重启

因为容器本身就是可移植的，所以 kubernetes 容器集群也能跑在私有云、公有云或者混合云上面。

kubernetes 让应用的集群管理变得简单，只要通过一行命令，就能快速搭建包含了前后端的完整系统集群。

```
kubectl create -f config-file.yml
```

## 安装 kubernetes

## kubernetes 的概念

### Pods

kubernetes 中，最基本的管理单位是 pod 而不是 container。pod 由一个或者多个容器组成，一般运行着相同的应用。一个 pod 中的所有容器都必须运行在同一台机器上，共享网络空间（network namespace）和存储 （volume）。

pod 可以近似类比传统模式下的主机：一些相关的应用组合起来，实现一个逻辑上的功能。

记住：pod 不是持久的，也就是说在使用过程中会被删除和创建，只要保证 service （下文会介绍到）保持稳定就行。而持久化的功能可以通过共享的 volume 来实现，这样即使 pod 被不断删除创建也能保证数据的完整。

### Replication Controllers

RC 保证在任意时刻都有指定数量的 pod 副本在运行。比如如果我们创建了一个 rc，指定某个 pod 运行 3 份：

- 刚开始的时候，rc 发现集群中没有这个 pod，它会创建 3 个 pod
- 当某个 pod 不响应或者被删除了，它会检测到这个变化，然后新建一个正常的 pod（替换错误状态的 pod），保证系统有 3 个 pod 在运行
- 当错误的 pod 回复正常，或者用户又手动添加了几个 pod，rc 也会检测到，它会通过删除 pod 来保证只有 3 份副本在运行

![](http://dockerone.com/uploads/article/20151230/5e2bad1a25e33e2d155da81da1d3a54b.gif)

创建 replication controller 的时候，需要几个参数：

- pod 的模板：用来创建应用的 pod 模板，在需要的时候根据这个模板创建 pod
- replicas 副本数：要达到的目标状态，pod 需要运行几份
- label：需要监控的 pod 的标签，通过比较 replicas 变量的值和标签实际发现的 pod 数量确定是否在目标状态

### Services

service，也就是服务，是实际环境中对外提供服务的业务抽象。一个服务后端可能有多个容器提供支持，但是对外通过服务这一层只有一个接口，通过转发用户的请求到后端的应用上来实现。这样做的好处是：对外隐藏了后端的细节，使得扩展和维护变得更容易。

在 kubernetes 中，service 后面对应的是 pod，这种对应关系是通过 label 来匹配的。每个 service 会配置 label selector 对后端的 pod 进行选择，因此这种关系是动态的，pod 的添加和删除都会自动被关联进来。

service 是单独存在的，不必和 pod 或者 replication controller 绑定。即使后端没有 pod，service 的存在也是合法的，当对应的 pod 出现时，会动态的绑定到 service 上。

### Labels

labels 就是标签的意思，主要用于过滤 pod、service 或者 replication controller，尤其是当集群的规模比较大的时候。

一个 label 就是一个键值对，用来表示用户自己定义的属性。给 pod 添加添加合理的标签会做到事半功倍的效果，增加可读性（通过 label 就能大致了解 pod 的功能）的同时也能方便查找对应的服务（service 和 replication controller 就是通过 label 来绑定后端 pod 的）。

利用 label 能够把 service 和 pod 做到松耦合：两者都可以单独存在，并且这种绑定关系是动态的。

## kubernetes 组件

kubernetes 整体上的框架是下面这样的，由多个不同的部分组成，下面将逐个讲解这些部分的功能。

![](https://blog.jetstack.io/images/k8s/architecture-small.png)

### kubectl

这是 kubernetes 提供的客户端程序，也是目前常用的和集群交互的方式。创建、查看、管理、删除、更新 pod、service、replication controller 都行，还有更多其他命令，可以查看帮助文档。

### etcd 集群

etcd 是 kubernetes 存放集群状态和配置的地方，这是集群状态同步的关键，所有节点都是从 etcd 中获取集群中其他机器状态的；集群中所有容器的状态也是放在这里的。

kubernetes 在 etcd 中存储的数据在 `/registry` 路径下，结构类似下面这样：

```
  /registry/minions
  /registry/namespaces
  /registry/pods
  /registry/ranges
  /registry/serviceaccounts
  /registry/services
  /registry/controllers
  /registry/events
```

### Master 组件

kubernetes 是典型的 master-slave 模式，master 是整个集群的大脑，负责控制集群的方方面面。

#### API server

对外提供 kubernetes API，也就是 kubernetes 对外的统一入口。封装了 kubernetes 所有的逻辑，通过 RESTful 的方式供客户端使用，kubectl 就是最常用到的客户端程序。

#### Scheduler

调度器：实现容器调度的组件，调度算法可以由用户自己实现。Scheduler 会收集并分析当前系统中所有 slave 节点的负载情况，在调度的时候作为决策的重要依据。

调度器监听 etcd 中 pods 目录的变化，当发现新的 pod 时，会利用调度算法把 pod 放到某个节点进行部署。可能的 scheduler 包括：

- random：随机调度算法
- round robin：轮询调度

#### Controller Manager Server

 集群中其他功能都是在 controller manager 中实现的，每个部分负责一个独立功能的管理。例如：

- endpoints controller
- node controller
- replication controller
- service controller
- ……

### slave 组件

#### kubelet

kubelet 是 slave 上核心的工作进程，负责容器和 pod 的实际管理工作（创建、删除等）以及和 master 通信，内容包括：

- 负责容器的创建、停止、删除等具体工作，也包括镜像下载、 volume 管理
- 从 etcd 中获取分配给自己的信息，根据其中的信息来控制容器以达到对应的目标状态
- 接受来自 master 的请求，汇报本节点的信息

#### kube-proxy

正如名字提示的一样，kube-proxy 为 pod 提供代理服务，用来实现 kubernetes 的 service 机制。每个 service 都会被分配一个虚拟 ip，当应用访问到 service ip 和端口的时候，会先发送到 kube-proxy，然后才转发到机器上的容器服务。

## 参考资料

- [kubernetes 系统架构简介](http://www.infoq.com/cn/articles/Kubernetes-system-architecture-introduction)
- [introduction to kubernetes](http://www.slideshare.net/rajdeep/introduction-to-kubernetes)
