---
layout: post
title: "CNI：容器网络接口"
excerpt: "CNI 是这样一个标准，它旨在为容器平台提供网络的标准化。不同的容器平台（比如 kubernetes、mesos 和 rkt）能够通过相同的接口调用不同的网络组件。"
categories: blog
tags: [kubernetes, cni, container, network]
comments: true
share: true
---

## CNI 简介

不管是 docker 还是 kubernetes，在网络方面目前都没有一个完美的、终极的、普适性的解决方案，不同的用户和企业因为各种原因会使用不同的网络方案。目前存在网络方案 flannel、calico、openvswitch、weave、ipvlan等，而且以后一定会有其他的网络方案，这些方案接口和使用方法都不相同，而不同的容器平台都需要网络功能，它们之间的适配如果没有统一的标准，会有很大的工作量和重复劳动。

CNI 就是这样一个标准，它旨在为容器平台提供网络的标准化。不同的容器平台（比如目前的 kubernetes、mesos 和 rkt）能够通过相同的接口调用不同的网络组件。

CNI(Conteinre Network Interface) 是 google 和 CoreOS 主导制定的容器网络标准，它 本身并不是实现或者代码，可以理解成一个协议。这个标准是在 [rkt 网络提议](https://docs.google.com/a/coreos.com/document/d/1PUeV68q9muEmkHmRuW10HQ6cHgd4819_67pIxDRVNlM/edit#heading=h.ievko3xsjwxd) 的基础上发展起来的，综合考虑了灵活性、扩展性、ip 分配、多网卡等因素。

这个协议连接了两个组件：容器管理系统和网络插件。它们之间通过 JSON 格式的文件进行通信，实现容器的网络功能。具体的事情都是插件来实现的，包括：创建容器网络空间（network namespace）、把网络接口（interface）放到对应的网络空间、给网络接口分配 IP 等等。

![](https://cdn.thenewstack.io/media/2016/09/Chart_Container-Network-Interface-Drivers.png)

关于网络，docker 也提出了 CNM 标准，它要解决的问题和 CNI 是重合的，也就是说目前两者是竞争关系。目前 CNM 只能使用在 docker 中，而 CNI 可以使用在任何容器运行时。CNM 主要用来实现 docker 自身的网络问题，也就是 `docker network` 子命令提供的功能。

## 官方网络插件

所有的标准和协议都要有具体的实现，才能够被大家使用。CNI 也不例外，目前官方在 github 上维护了同名的 [CNI](https://github.com/containernetworking/cni) 代码库，里面已经有很多可以直接拿来使用的 CNI 插件。

官方提供的插件目前分成三类：main、meta 和 ipam。main 是主要的实现了某种特定网络功能的插件；meta 本身并不会提供具体的网络功能，它会调用其他插件，或者单纯是为了测试；ipam 是分配 IP 地址的插件。

ipam 并不提供某种网络功能，只是为了灵活性把它单独抽象出来，这样不同的网络插件可以根据需求选择 ipam，或者实现自己的 ipam。

这些插件的功能说明如下：

- main
    - loopback：这个插件很简单，负责生成 `lo` 网卡，并配置上 `127.0.0.1/8` 地址
    - bridge：和 docker 默认的网络模型很像，把所有的容器连接到虚拟交换机上
    - macvlan：使用 macvlan 技术，从某个物理网卡虚拟出多个虚拟网卡，它们有独立的 ip 和 mac 地址
    - ipvlan：和 macvlan 类似，区别是虚拟网卡有着相同的 mac 地址
    - ptp：通过 veth pair 在容器和主机之间建立通道
- meta
    - flannel：结合 bridge 插件使用，根据 flannel 分配的网段信息，调用 bridge 插件，保证多主机情况下容器
- ipam
    - host-local：基于本地文件的 ip 分配和管理，把分配的 IP 地址保存在文件中
    - dhcp：从已经运行的 DHCP 服务器中获取 ip 地址

## 接口参数

网络插件是独立的可执行文件，被上层的容器管理平台调用。网络插件只有两件事情要做：把容器加入到网络以及把容器从网络中删除。调用插件的数据通过两种方式传递：环境变量和标准输入。一般插件需要三种类型的数据：容器相关的信息，比如 ns 的文件、容器 id 等；网络配置的信息，包括网段、网关、DNS 以及插件额外的信息等；还有就是 CNI 本身的信息，比如 CNI 插件的位置、添加网络还是删除网络。

我们来看一下为容器添加网络是怎么工作的，删除网络和它过程一样。

**把容器加入到网络**

调用插件的时候，这些参数会通过环境变量进行传递：

- `CNI_COMMAND`：要执行的操作，可以是 `ADD`（把容器加入到某个网络）、`DEL`（把容器从某个网络中删除）
- `CNI_CONTAINERID`：容器的 ID，比如 ipam 会把容器 ID 和分配的 IP 地址保存下来。可选的参数，但是推荐传递过去。需要保证在管理平台上是唯一的，如果容器被删除后可以循环使用
- `CNI_NETNS`：容器的 network namespace 文件，访问这个文件可以在容器的网络 namespace 中操作
- `CNI_IFNAME`：要配置的 interface 名字，比如 `eth0`
- `CNI_ARGS`：额外的参数，是由分号`;`分割的键值对，比如 "FOO=BAR;hello=world"
- `CNI_PATH`：CNI 二进制查找的路径列表，多个路径用分隔符 `:` 分隔

网络信息主要通过标准输入，作为 JSON 字符串传递给插件，必须的参数包括：

- `cniVersion`：CNI 标准的版本号。因为 CNI 在演化过程中，不同的版本有不同的要求
- `name`：网络的名字，在集群中应该保持唯一
- `type`：网络插件的类型，也就是 CNI 可执行文件的名称
- `args`：额外的信息，类型为字典
- `ipMasq`：是否在主机上为该网络配置 IP masquerade
- `ipam`：IP 分配相关的信息，类型为字典
- `dns`：DNS 相关的信息，类型为字典

插件接到这些数据，从输入和环境变量解析到需要的信息，根据这些信息执行程序逻辑，然后把结果返回给调用者，返回的结果中一般包括这些参数：

- IPs assigned to the interface：网络接口被分配的 ip，可以是 IPv4、IPv6 或者都有
- DNS 信息：包含 nameservers、domain、search domains 和其他选项的字典

CNI 协议的内容还在不断更新，请到[官方文档](https://github.com/containernetworking/cni/blob/master/SPEC.md)获取当前的信息。

## CNI 的特性

CNI 作为一个协议/标准，它有很强的扩展性和灵活性。如果用户对某个插件有额外的需求，可以通过输入中的 `args` 和环境变量 `CNI_ARGS` 传输，然后在插件中实现自定义的功能，这大大增加了它的扩展性；CNI 插件把 main 和 ipam 分开，用户可以自由组合它们，而且一个 CNI 插件也可以直接调用另外一个 CNI 插件，使用起来非常灵活。

如果要实现一个继承性的 CNI 插件也不复杂，可以编写自己的 CNI 插件，根据传入的配置调用 main 中已经有的插件，就能让用户自由选择容器的网络。

## 在 kubernetes 中的使用

CNI 目前已经在 kubernetes 中开始使用，也是目前官方推荐的网络方案，具体的配置方法可以参考[kubernetes 官方文档](https://kubernetes.io/docs/concepts/cluster-administration/network-plugins/#cni)。

kubernetes 使用了 CNI 网络插件之后，工作过程是这样的：

- kubernetes 先创建 pause 容器生成对应的 network namespace
- 调用网络 driver（因为配置的是 CNI，所以会调用 CNI 相关代码）
- CNI driver 根据配置调用具体的 cni 插件
- cni 插件给 pause 容器配置正确的网络
- pod 中其他的容器都是用 pause 的网络

## 参考资料

- [CNI spec 文档](https://github.com/containernetworking/cni/blob/master/SPEC.md)
- [Linux Network namespaces and CNI](http://murat1985.github.io/kubernetes/cni/2016/05/14/netns-and-cni.html)
- [The container networking landscape: cni from coreos and cnm from docker](https://thenewstack.io/container-networking-landscape-cni-coreos-cnm-docker/)
- [CNI and OCI](http://linuxplumbersconf.org/2015/ocw/system/presentations/3357/original/LPC_-_CNI_and_OCI.pdf)
