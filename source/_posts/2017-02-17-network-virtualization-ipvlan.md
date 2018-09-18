---
layout: post
title: "linux 网络虚拟化： ipvlan"
excerpt: "IPVlan 和 macvlan 类似，都是从一个主机接口虚拟出多个虚拟网络接口。一个重要的区别就是所有的虚拟接口都有相同的 macv 地址，而拥有不同的 ip 地址。"
categories: blog
tags: [network, container, linux, ipvlan]
comments: true
share: true
---

## 简介

IPVlan 和 macvlan 类似，都是从一个主机接口虚拟出多个虚拟网络接口。一个重要的区别就是所有的虚拟接口都有相同的 macv 地址，而拥有不同的 ip 地址。因为所有的虚拟接口要共享 mac 地址，所有有些需要注意的地方：

- DHCP 协议分配 ip 的时候一般会用 mac 地址作为机器的标识。这个情况下，客户端动态获取 ip 的时候需要配置唯一的 ClientID 字段，并且 DHCP server 也要正确配置使用该字段作为机器标识，而不是使用 mac 地址

[ipvlan](https://lwn.net/Articles/620087/) 是 linux kernel 比较新的特性，linux kernel 3.19 开始支持 ipvlan，但是比较稳定推荐的版本是 >=4.2（因为 docker 对之前版本的支持有 bug）。



## 两种模式

ipvlan 有两种不同的模式：L2 和 L3。一个父接口只能选择一种模式，依附于它的所有虚拟接口都运行在这个模式下，不能混用模式。

### L2 模式

ipvlan L2 模式和 macvlan bridge 模式工作原理很相似，父接口作为交换机来转发子接口的数据。同一个网络的子接口可以通过父接口来转发数据，而如果想发送到其他网络，报文则会通过父接口的路由转发出去。

### L3 模式

L3 模式下，ipvlan 有点像路由器的功能，它在各个虚拟网络和主机网络之间进行不同网络报文的路由转发工作。只要父接口相同，即使虚拟机/容器不在同一个网络，也可以互相 ping 通对方，因为 ipvlan 会在中间做报文的转发工作。

![](http://hicu.be/wp-content/uploads/2016/03/linux-ipvlan-l3-mode-1.png)

L3 模式下的虚拟接口**不会接收到多播或者广播的报文**，为什么呢？这个模式下，所有的网络都会发送给父接口，所有的 ARP 过程或者其他多播报文都是在底层的父接口完成的。需要注意的是：外部网络默认情况下是不知道 ipvlan 虚拟出来的网络的，如果不在外部路由器上配置好对应的路由规则，ipvlan 的网络是不能被外部直接访问的。

## 实验

实验环境：

```bash
➜  ~ uname -a
Linux cizixs-ThinkPad-T450 4.4.0-57-generic #78-Ubuntu SMP Fri Dec 9 23:50:32 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
➜  ~ lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 16.04.2 LTS
Release:	16.04
Codename:	xenial
```

先创建两个用做测试的 network namespace，关于 namespace 可参考[之前的文章](http://cizixs.com/2017/02/10/network-virtualization-network-namespace)：

```bash
➜  ~ sudo ip netns add net1
➜  ~ sudo ip netns add net2
```

然后创建出 ipvlan 的虚拟网卡接口，因为 L2 和 macvlan 功能相同，我们这里测试一下 L3 模式。创建 ipvlan 虚拟接口的命令和 macvlan 格式相同：

```bash
➜  ~ sudo ip link add ipv1 link eth0 type ipvlan mode l3
➜  ~ sudo ip link add ipv2 link eth0 type ipvlan mode l3
```

把 ipvlan 接口放到前面创建好的 namespace 中：

```bash
➜  ~ sudo ip link set ipv1 netns net1
➜  ~ sudo ip link set ipv2 netns net2
➜  ~ sudo ip netns exec net1 ip link set ipv1 up
➜  ~ sudo ip netns exec net2 ip link set ipv2 up
```

给两个虚拟网卡接口配置上不同网络的 ip 地址，并配置好路由项：

```bash
➜  ~ sudo ip netns exec net1 ip addr add 10.0.1.10/24 dev ipv1
➜  ~ sudo ip netns exec net2 ip addr add 192.168.1.10/24 dev ipv2
➜  ~ sudo ip netns exec net1 ip route add default dev ipv1
➜  ~ sudo ip netns exec net2 ip route add default dev ipv2
```

测试两个网络的连通性：

```bash
➜  ~ sudo ip netns exec net1 ping -c 3 192.168.1.10
PING 192.168.1.10 (192.168.1.10) 56(84) bytes of data.
64 bytes from 192.168.1.10: icmp_seq=1 ttl=64 time=0.053 ms
64 bytes from 192.168.1.10: icmp_seq=2 ttl=64 time=0.035 ms
64 bytes from 192.168.1.10: icmp_seq=3 ttl=64 time=0.036 ms

--- 192.168.1.10 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1998ms
rtt min/avg/max/mdev = 0.035/0.041/0.053/0.009 ms
```

## docker ipvlan 网络

docker（v1.13） 也在开发对 ipvlan 的支持，不过目前只是 experiment 阶段。你可以安装最新版的 docker 来进行测试：

```bash

# 首先是创建 ipvlan 的网络，这和 macvlan 网络的操作基本一致，只是把 driver 修改成 `ipvlan`，然后选项中通过 `ipvlan_mode=l3` 设置 ipvlan 的工作模式：

docker network  create  -d ipvlan \
    --subnet=192.168.30.0/24 \
    -o parent=eth0 \
    -o ipvlan_mode=l3 ipvlan30

# 启动两个容器，可以发现在同一个 ipvlan 的两个容器可以互相 ping 通
docker run --net=ipvlan30 -it --name ivlan_test3 --rm alpine /bin/sh
docker run --net=ipvlan30 -it --name ivlan_test4 --rm alpine /bin/sh

# 再创建另外一个 ipvlan 网络，和前面的网络不在同一个广播域
docker network  create  -d ipvlan \
    --subnet=192.168.110.0/24 \
    -o parent=eth0 \
    -o ipvlan_mode=l3 ipvlan110

# 在新建的网络中运行容器，可以发现可以 ping 同前面网络中的容器
docker run --net=ipvlan30 -it --name ivlan_test3 --rm alpine /bin/sh
```

这里只介绍了最基本的 ipvlan 在 docker 中的使用，更多可以参考 [docker 官方文档](https://github.com/docker/docker/blob/master/experimental/vlan-networks.md)。

## ipvlan 还是 macvlan？

ipvlan 和 macvlan 两个虚拟网络模型提供的功能，看起来差距并不大，那么什么时候需要用到 ipvlan 呢？要回答这个问题，我们先来看看 macvlan 存在的不足：

- 需要大量 mac 地址。每个虚拟接口都有自己的 mac 地址，而网络接口和交换机支持的 mac 地址有支持的上限
- 无法和 802.11(wireless) 网络一起工作

对应的，如果你遇到一下的情况，请考虑使用 ipvlan：

- 父接口对 mac 地址数目有限制，或者在 mac 地址过多的情况下会造成严重的性能损失
- 工作在无线网络中
- 希望搭建比较复杂的网络拓扑（不是简单的二层网络和 VLAN），比如要和 BGP 网络一起工作

## 参考资料

- [Configuring Macvlan and Ipvlan Linux Networking](http://networkstatic.net/configuring-macvlan-ipvlan-linux-networking/)
- [Linux Kernel ipvlan documentation](https://www.kernel.org/doc/Documentation/networking/ipvlan.txt)
- [Docker IPVlan Network Driver Doc](https://github.com/docker/docker/blob/master/experimental/vlan-networks.md)
- [ipvlan practice and implementation](http://hustcat.github.io/ipvlan-practice-and-implementation/)
- [Macvlan and IPvlan basics](https://sreeninet.wordpress.com/2016/05/29/macvlan-and-ipvlan/)
