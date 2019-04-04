---
layout: post
title: "linux 上实现 vxlan 网络"
excerpt: "Linux 对 vxlan 协议的支持时间并不久，2012 年 Stephen Hemminger 才把相关的工作合并到 kernel 中，并最终出现在 kernel 3.7.0 版本。"
categories: blog
tags: [linux, vxlan, network, sdn]
comments: true
share: true
---

## linux 上 vxlan 简介

[vxlan 协议的介绍文章](http://cizixs.com/2017/09/25/vxlan-protocol-introduction)主要介绍了 vxlan 协议的理论知识，从产生的背景到报文的格式等等，和所有的计算机知识一样，理论必须结合实践理解才能更深刻，这篇文章我们就讲讲在 linux 机器上怎么手动搭建 vxlan overlay 网络。

Linux 对 vxlan 协议的支持时间并不久，2012 年 Stephen Hemminger 才把相关的工作合并到 kernel 中，并最终出现在 kernel 3.7.0 版本。为了稳定性和很多的功能，你可以会看到某些软件推荐在 3.9.0 或者 3.10.0 以后版本的 kernel 上使用 vxlan。

**NOTE**：如果可以，尽量使用比较新版本的 kernel，以免出现因为内核版本太低导致功能或者性能上的问题。

本文的实验环境：

- MacBook Pro: 2.6 GHz Intel Core i5
- virtualbox 5.1.8
- vagrant 2.0.0
- OS: ubuntu 16.04 
- Linux Kernel: 4.4.0-83-generic

我用 vagrant 启动了三台虚拟机，它们之间通信的 IP 地址（也就是 underlay 网络）为：

- `192.168.8.100`
- `192.168.8.101`
- `192.168.8.102`

而要创建的 overlay 网络网段为 `10.20.1.0/24`，实验目的就是 vxlan 能够通过 overlay IP 互相连通。

## 实验

### 点对点的 vxlan

我们先来搭建一个最简单的 vxlan 网络，两台机器构成一个 vxlan 网络，每台机器上有一个 vtep，vtep 通过它们的 IP 互相通信。这次实验完成后的网络结构如下图所示：

![](https://ws1.sinaimg.cn/large/006tKfTcgy1fjy54027bgj31hc0u0tde.jpg)

首先使用 `ip` 命令创建我们的 vxlan interface：

```
$ ip link add vxlan0 type vxlan \
    id 42 \
    dstport 4789 \
    remote 192.168.8.101 \
    local 192.168.8.100 \
    dev enp0s8 
```

上面这条命令创建一个名字为 `vxlan0`，类型为 `vxlan` 的网络 interface，后面是 vxlan interface 需要的参数：

- `id 42`：指定 VNI 的值，这个值可以在 1 到 2^24 之间
- `dstport`：vtep 通信的端口，linux 默认使用 8472（为了保持兼容，默认值一直没有更改），而 IANA 分配的端口是 4789，所以我们这里显式指定了它的值
- `remote 192.168.8.101`：对方 vtep 的地址，类似于点对点协议
- `local 192.168.8.100`：当前节点 vtep 要使用的 IP 地址
- `dev enp0s8`：当节点用于 vtep 通信的网卡设备，用来读取 IP 地址。注意这个参数和 `local` 参数含义是相同的，在这里写出来是为了告诉大家有两个参数存在

执行完之后，系统就会创建一个名字为 `vxlan0` 的 interface，可以用 `ip -d link` 查看它的详细信息：

```
root@node1:~# ip -d link show dev vxlan0
4: vxlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether ba:d4:0a:f8:41:7c brd ff:ff:ff:ff:ff:ff promiscuity 0
    vxlan id 42 remote 192.168.8.101 dev enp0s8 srcport 0 0 dstport 4789 ageing 300 addrgenmode eui64
```

**NOTE**：`ip` 命令来自于 `iproute2` 软件包，它主要用来做网络相关的操作。

接下来，为刚创建的 interface 配置 IP 地址并启用它：

```
$ ip addr add 10.20.1.2/24 dev vxlan0
$ ip link set vxlan0 up
```

执行结束之后，会发现路由表项多了下面的内容，所有 `10.20.1.0/24` 网段的 IP 地址要通过 vxlan0 来转发：

```
root@node0:~# ip route
10.20.1.0/24 dev vxlan0  proto kernel  scope link  src 10.20.1.2
```

同时，vxlan0 fdb 表项中的内容如下：

```
root@node1:~# bridge fdb
00:00:00:00:00:00 dev vxlan0 dst 192.168.8.101 via enp0s8 self permanent
```

这个表项的意思是说，默认的而 vtep 对端地址为 `192.168.8.101`，换句话说，如果接收到的报文添加上 vxlan 头部之后都会发到 `192.168.8.101`。

在另外一台虚拟机（`192.168.8.101`）上也进行相同的配置，要保证 VNI 也是 42，dstport 也是 4789，并修改 vtep 的地址和 remote IP 地址到相应的值。测试两台 vtep 的连通性：

```
root@node0:~# ping -c 3 10.20.1.3
PING 10.20.1.3 (10.20.1.3) 56(84) bytes of data.
64 bytes from 10.20.1.3: icmp_seq=1 ttl=64 time=1.84 ms
64 bytes from 10.20.1.3: icmp_seq=2 ttl=64 time=0.462 ms
64 bytes from 10.20.1.3: icmp_seq=3 ttl=64 time=0.427 ms

--- 10.20.1.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 0.427/0.911/1.844/0.659 ms
```

这种点对点的 vxlan 网络只能两两通信，实际用处不大。下面我们介绍多节点怎么组成 vxlan 网络进行通信。

### 多播模式的 vxlan

如果 vxlan 要使用多播模式，那么底层的网络结构需要支持多播的功能，幸运的是，virtualbox 本身就支持多播，不需要额外设置。

要组成同一个 vxlan 网络，vtep 必须能感知到彼此的存在。多播组本来的功能就是把网络中的某些节点组成一个虚拟的组，所以 vxlan 最初想到用多播来实现是很自然的事情。

这个实验和前面一个非常相似，只不过主机之间不是点对点的连接，而是通过多播组成一个虚拟的整体。最终的网络架构也很相似（为了简单图中只有两个主机，但这个模型可以容纳多个主机组成 vxlan 网络）：

![](https://ws1.sinaimg.cn/large/006tKfTcgy1fjy54jqv9xj31hc0u0n2d.jpg)

可能听起来要加入同一个多播组挺复杂的，但实际上非常简单，先上命令：

```
$ ip link add vxlan0 type vxlan \
    id 42 \
    dstport 4789 \
    group 239.1.1.1 \
    dev enp0s8 
$ ip addr add 10.20.1.2/24 dev vxlan0
$ ip link set vxlan0 up
```

这里最重要的参数是 `group 239.1.1.1` 表示把 vtep 加入到这个多播组。关于多播的原理和使用不是这篇文章的重点，这里选择的多播 IP 地址也没有特殊的含义，关于多播的内容可以自行了解。

运行上面的命令之后，一样添加了对应的路由，不同是的 fdb 表项：

```
root@node0:~# bridge fdb
00:00:00:00:00:00 dev vxlan0 dst 239.1.1.1 via enp0s8 self permanent
```

这里默认表项的 dst 字段的值变成了多播地址 `239.1.1.1`，而不是之前对方的 vtep 地址。同理给所有需要通信的节点进行上述配置，可以验证他们能通过 10.20.1.0/24 网络互相访问。

我们来分析这个模式下 vxlan 通信的过程：

在配置完成之后，vtep 通过 IGMP 加入同一个多播网络 `239.1.1.1`。

1. 发送 ping 报文到 10.20.1.3，查看路由表，报文会从 vxlan0 发出去
2. 内核发现 vxlan0 的 IP 是 10.20.1.2/24，和目的 IP 在同一个网段，所以在同一个局域网，需要知道对方的 MAC 地址，因此会发送 ARP 报文查询
3. ARP 报文源 MAC 地址为 vxlan0 的 MAC 地址，目的 MAC 地址为全 1 的广播地址
4. vxlan 根据配置（VNI 42）添加上头部
5. 因为不知道对方 vtep 在哪台主机上，根据配置，vtep 会往多播地址 239.1.1.1 发送多播报文
6. 多播组中所有的主机都会受到这个报文，内核发现是 vxlan 报文，会根据 VNI 发送给对应的 vtep
7. vtep 去掉 vxlan 头部，取出真正的 ARP 请求报文。同时 vtep 会记录 `<源 MAC 地址 - vtep 所在主机 IP 地址>` 信息到 fdb 表中
8. 如果发现 ARP 不是发送给自己的，直接丢弃；如果是发送给自己的，则生成 ARP 应答报文
9. 应答报文目的 MAC 地址是发送方 vtep 的 MAC 地址，而且 vtep 已经通过源报文学习到了 vtep 所在的主机，因此会直接单播发送给目的 vtep。因此 vtep 不需要多播，就能填充所有的头部信息
10. 应答报文通过 underlay 网络直接返回给发送方主机，发送方主机根据 VNI 把报文转发给 vtep，vtep 解包取出 ARP 应答报文，添加 arp 缓存到内核。并根据报文学习到目的 vtep 所在的主机地址，添加到 fdb 表中
11. vtep 已经知道了通信需要的所有信息，后续 ICMP 的 ping 报文都是单播进行的

在这个过程中，在主机上抓包更容易看到通信的具体情况，下面是 ARP 请求报文的详情：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1fjyb6b5ybdj30x20lq11q.jpg)

可以看到 vxlan 报文可以分为三块：

- 里层（图中最下面）是 overlay 网络中实体看到的报文（比如这里的 ARP 请求），它们和经典网络的通信报文没有任何区别，除了因为 MTU 导致有些报文比较小
- 然后是 vxlan 头部，我们最关心的字段 VNI 确实是 42
- 最外层（图中最上面）是 vtep 所在主机的通信报文头部。可以看到这里 IP 地址为多播 `239.1.1.1`，目的 MAC 地址也是多播对应的地址

而 ARP 应答报文不是多播而是单播的事实也能看出来：

![](https://ws1.sinaimg.cn/large/006tKfTcgy1fjybddu0boj31kw11ekaq.jpg)

从上面的通信过程，可以看出不少信息：

- 多播其实就相当于 vtep 之间的广播，报文会发给所有的 vtep，但是只有一个会做出应答
- vtep 会通过接收到的报文学习 `MAC - VNI - Vtep IP` 的信息，减少后续不必要的多播报文
- 对于 overlay 网络中的通信实体来说，整个通信过程对它们的透明的，它们认为自己的通信过程和经典网络没有区别

通信结束之后，可以在主机上看到保存的 ARP 缓存：

```
root@node0:~# ip neigh
10.20.1.3 dev vxlan0 lladdr d6:d9:cd:0a:a4:28 STALE
```

以及 vtep 需要的 fdb 缓存：

```
root@node0:~# bridge fdb
00:00:00:00:00:00 dev vxlan0 dst 239.1.1.1 via enp0s8 self permanent
d6:d9:cd:0a:a4:28 dev vxlan0 dst 192.168.8.101 self
```

### 利用 bridge 来接入容器

尽管上面的方法能够通过多播实现自动化的 overlay 网络构建，但是通信的双方只有 vtep，在实际的生产中，每台主机上都有几十台甚至上百台的虚拟机或者容器需要通信，因此我们需要找到一种方法能够把这些通信实体组织起来。

在 linux 中把同一个网段的 interface 组织起来正是网桥（bridge，或者 switch，这两个名称等价）的功能，因此这部分我们介绍如何用网桥把多个虚拟机或者容器放到同一个 vxlan overlay 网络中。最终实现的网络架构如下图所示：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1fjy5517ktoj31hc0u0n39.jpg)

因为创建虚拟机或者容器比较麻烦，我们用 network namespace 来模拟，从理论上它们是一样的。关于 network namespace 和 veth pair 的基础知识，请参考我[之前的介绍文章](http://cizixs.com/2017/02/10/network-virtualization-network-namespace)。

对于每个容器/虚拟机，我们创建一个 network namespace，并通过一对 veth pair 把容器中的 eth0 网络连接到网桥上。同时 vtep 也会放到网桥上，以便能够对报文进行 vxlan 相关的处理。

首先我们创建 vtep interface，使用的是多播模式：

创建 vtep interface：
```
$ ip link add vxlan0 type vxlan \
    id 42 \
    dstport 4789 \
    group 239.1.1.1 \
    local 192.168.8.100 \
    dev enp0s8 
```

然后创建网桥 `br0`，把 vtep interface 绑定到上面：

```
$ ip link add br0 type bridge
$ ip link set vxlan100 master bridge
$ ip link set vxlan100 up
$ ip link set br0 up
```

下面模拟把容器加入到网桥的操作，创建一个 network namespace，和一对 veth pair：

```
ip netns add container1

# 创建 veth pair，并把一端加到网桥上
ip link add veth0 type veth peer name veth1
ip link set dev veth0 master br0
ip link set dev veth0 up

# 配置容器内部的网络和 IP
ip link set dev veth1 netns container1
ip netns exec container1 ip link set lo up

ip netns exec container1 ip link set veth1 name eth0
ip netns exec container1 ip addr add 10.20.1.2/24 dev eth0
ip netns exec container1 ip link set eth0 up
```

为了方便操作，我把上面的过程写成了一个[脚本](https://gist.github.com/cizixs/cfee4e8885df40c92f04b18807d1fb9d)。

<script src="https://gist.github.com/cizixs/cfee4e8885df40c92f04b18807d1fb9d.js"></script>

使用这个脚本，下面的命令就能方便地创建另外一个容器：

```
$ ./set_container br0 container4 10.20.1.4/24 
```

用同样的方法在另外一台主机上配置 vxlan 网络，添加 IP 为 10.20.1.3/24 的容器，并测试它们的连通性。

容器通信过程和前面的实验类似，只不过这里容器发出的 ARP 报文会被网桥转发给 `vxlan0`，然后 `vxlan0` 添加 vxlan 头部通过多播来找到对方的 MAC 地址。

从逻辑上可以认为，在 `vxlan1` 的帮助下同一个 vxlan overlay 网络中的容器是连接到同一个网桥上的，示意图如下：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1fjy55dn4shj31hc0u0ag5.jpg)

多播实现很简单，不需要中心化的控制。但是不是所有的网络都支持多播，而且需要事先规划多播组和 VNI 的对应关系，在 overlay 网络数量比较多时也会很麻烦，多播也会导致大量的无用报文在网络中出现。现在很多云计算的网络都会通过自动化的方式来发现 vtep 和 MAC 信息，也就是自动构建 vxlan 网络。下面的几个部分，我们来解开自动化 vxlan 网络的秘密。

### 手动维护 vtep 组

经过上面几个实验，我们来思考一下为什么要使用多播。因为对 overlay 网络来说，它的网段范围是分布在多个主机上的，因此传统 ARP 报文的广播无法直接使用。要想做到 overlay 网络的广播，必须把报文发送到所有 vtep 在的节点，这才引入了多播。

如果有一种方法能够不通过多播，能把 overlay 的广播报文发送给所有的 vtep 主机的话，也能达到相同的功能。当然在维护 vtep 网络组之前，必须提前知道哪些 vtep 要组成一个网络，以及这些 vtep 在哪些主机上。

Linux 的 vxlan 模块也提供了这个功能，而且实现起来并不复杂。创建 vtep interface 的时候不使用 `remote` 或者 `group` 参数就行：

```
$ ip link add vxlan0 type vxlan \
    id 42 \
    dstport 4789 \
    dev enp0s8 
```

这个 vtep interface 创建的时候没有指定多播地址，当第一个 ARP 请求报文发送时它也不知道要发送给谁。但是我们可以手动添加默认的 FDB 表项，比如：

```
$ bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst 192.168.8.101
$ bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst 192.168.8.102
```

这样的话，如果不知道对方 VTEP 的地址，就会往选择默认的表项，发到 `192.168.8.101` 和 `192.168.8.102`，相当于手动维护了一个 vtep 的多播组。

在所有的节点的 vtep 上更新对应的 fdb 表项，就能实现 overlay 网络的连通。整个通信流程和多播模式相同，唯一的区别是，vtep 第一次会给所有的组内成员发送单播报文，当然也只有一个 vtep 会做出应答。

使用一些自动化工具，定时更新 FDB 表项，就能动态地维护 VTEP 的拓扑结构。

这个方案解决了在某些 underlay 网络中不能使用多播的问题，但是并没有解决多播的另外一个问题：每次要查找 MAC 地址要发送大量的无用报文，如果 vtep 组节点数量很大，那么每次查询都发送 N 个报文，其中只有一个报文真正有用。

### 手动维护 fdb 表

如果提前知道目的容器 MAC 地址和它所在主机的 IP 地址，也可以通过更新 fdb 表项来减少广播的报文数量。


创建 vtep 的命令：

```
$ ip link add vxlan0 type vxlan \
    id 42 \
    dstport 4789 \
    dev enp0s8 \
    nolearning
```

这次我们添加了 `nolearning` 参数，这个参数告诉 vtep 不要通过收到的报文来学习 fdb 表项的内容，因为我们会自动维护这个列表。

然后可以添加 fdb 表项告诉 vtep 容器 MAC 对应的主机 IP 地址：

```
$ bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst 192.168.8.101
$ bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst 192.168.8.102
$ bridge fdb append 52:5e:55:58:9a:ab dev vxlan0 dst 192.168.8.101
$ bridge fdb append d6:d9:cd:0a:a4:28 dev vxlan0 dst 192.168.8.102
```

如果知道了对方的 MAC 地址，vtep 搜索 fdb 表项就知道应该发送到哪个对应的 vtep 了。需要注意是的，这个情况还是需要默认的表项（那些全零的表项），在不知道容器 IP 和 MAC 对应关系的时候通过默认方式发送 ARP 报文去查询对方的 MAC 地址。

需要注意的是，和上一个方法相比，这个方法并没有任何效率上的改进，只是把自动学习 fdb 表项换成了手动维护（当然实际情况一般是自动化程序来维护），第一次发送 ARP 请求还是会往 vtep 组发送大量单播报文。

当时这个方法给了我们很重要的提示：如果实现知道 vxlan 网络的信息，**vtep 需要的信息都是可以自动维护的，而不需要学习**。

### 手动维护 ARP 表

除了维护 fdb 表，arp 表也是可以维护的。如果能通过某个方式知道容器的 IP 和 MAC 地址对应关系，只要更新到每个节点，就能实现网络的连通。

但是这里有个问题，我们需要维护的是每个容器里面的 ARP 表项，因为最终通信的双方是容器。到每个容器里面（所有的 network namespace）去更新对应的 ARP 表，是件工作量很大的事情，而且容器的创建和删除还是动态的，。linux 提供了一个解决方案，vtep 可以作为 arp 代理，回复 arp 请求，也就是说只要 vtep interface 知道对应的 `IP - MAC` 关系，在接收到容器发来的 ARP 请求时可以直接作出应答。这样的话，我们只需要更新 vtep interface 上 ARP 表项就行了。 

创建 vtep interface 需要加上 `proxy` 参数：

```
$ ip link add vxlan0 type vxlan \
    id 42 \
    dstport 4789 \
    dev enp0s8 \
    nolearning \
    proxy
```

这条命令和上部分相比多了 `proxy` 参数，这个参数告诉 vtep 承担 ARP 代理的功能。如果收到 ARP 请求，并且自己知道结果就直接作出应答。

当然我们还是要手动更新 fdb 表项来构建 vtep 组，

```
$ bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst 192.168.8.101
$ bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst 192.168.8.102
$ bridge fdb append 52:5e:55:58:9a:ab dev vxlan0 dst 192.168.8.101
$ bridge fdb append d6:d9:cd:0a:a4:28 dev vxlan0 dst 192.168.8.102
```

然后，还需要为 vtep 添加 arp 表项，所有要通信容器的 `IP - MAC `二元组都要加进去。
```
$ ip neigh add 10.20.1.3 lladdr d6:d9:cd:0a:a4:28 dev vxlan0
$ ip neigh add 10.20.1.4 lladdr 52:5e:55:58:9a:ab dev vxlan0
```

在要通信的所有节点配置完之后，容器就能互相 ping 通。当容器要访问彼此，并且第一次发送 ARP 请求时，这个请求并不会发给所有的 vtep，而是当前由当前 vtep 做出应答，大大减少了网络上的报文。

借助自动化的工具做到实时的表项（fdb 和 arp）更新，这种方法就能很高效地实现 overlay 网络的通信。

### 动态更新 arp 和 fdb 表项

尽管前一种方法通过动态更新 fdb 和 arp 表避免多余的网络报文，但是还有一个的问题：为了能够让所有的容器正常工作，所有可能会通信的容器都必须提前添加到 ARP 和 fdb 表项中。但并不是网络上所有的容器都会互相通信，所以**添加的有些表项（尤其是 ARP 表项）是用不到的**。

Linux 提供了另外一种方法，内核能够动态地通知节点要和哪个容器通信，应用程序可以订阅这些事件，如果内核发现需要的 ARP 或者 fdb 表项不存在，会发送事件给订阅的应用程序，这样应用程序从中心化的控制拿到这些信息来更新表项，做到更精确的控制。

要收到 L2（fdb）miss，必须要满足几个条件：

- 目的 MAC 地址未知，也就是没有对应的 fdb 表项
- fdb 中没有全零的表项，也就是说默认规则
- 目的 MAC 地址不是多播或者广播地址

要实现这种功能，创建 vtep 的时候需要加上额外的参数：

```
$ ip link add vxlan0 type vxlan \
    id 42 \
    dstport 4789 \
    dev enp0s8 \
    nolearning \
    proxy \
    l2miss \
    l3miss
```

这次多了两个参数 `l2miss` 和 `l3miss`：

- `l2miss`：如果设备找不到 MAC 地址需要的 vtep 地址，就发送通知事件
- `l3miss`：如果设备找不到需要 IP 对应的 MAC 地址，就发送通知事件

`ip monitor` 命令能做到这点，监听某个 interface 的事件，具体用法请参考 man 手册。

```
root@node0:~# ip monitor all dev vxlan0
```

如果从本节点容器 ping 另外一个节点的容器，就先发生 l3 miss，这是 l3miss 的通知事件，：

```
root@node0:~# ip monitor all dev vxlan0
[nsid current]miss 10.20.1.3  STALE
```

`l3miss` 是说这个 IP 地址，vtep 不知道它对应的 MAC 地址，因此要手动添加 arp 记录：

```
$ ip neigh replace 10.20.1.3 \
    lladdr b2:ee:aa:42:8b:0b \
    dev vxlan0 \
    nud reachable
```

上面这条命令设置的 `nud reachable` 参数意思是，这条记录有一个超时时间，系统发现它无效一段时间会自动删除。这样的好处是，不需要手动去删除它，删除后需要通信内核会再次发送通知事件。 `nud` 是 `Neighbour Unreachability Detection` 的缩写， 当然根据需要这个参数也可以设置成其他值，比如 `permanent`，表示这个记录永远不会过时，系统不会检查它是否正确，也不会删除它，只有管理员也能对它进行修改。

这时候还是不能正常通信，接着会出现 l2miss 的通知事件：

```
root@node0:~# ip monitor all dev vxlan0
[nsid current]miss lladdr b2:ee:aa:42:8b:0b STALE
```

类似的，这个事件是说不知道这个容器的 MAC 地址在哪个节点上，所以要手动添加 fdb 记录：

```
root@node0:~# bridge fdb add b2:ee:aa:42:8b:0b dst 192.168.8.101 dev vxlan0
```

在通信的另一台机器上执行响应的操作，就会发现两者能 ping 通了。

## 总结

上面提出的所有方案中，其中手动的部分都可以使用程序来自动完成，需要的信息一般都是从集中式的控制中心获取的，这也是大多数基于 vxlan 的 SDN 网络的大致架构。当然具体的实现不一定和某种方法相同，可能是上述方法的变形或者组合，但是设计思想都是一样的。

虽然上述的实验中，为了简化图中只有两台主机，而且只有一个 vxlan 网络，但是利用相同的操作很容易创建另外一个 vxlan 网络（必须要保证 vtep 的 VNI 值不同，如果使用多播，也要保证多播 IP 不同），如下图所示：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1fjy55xwqxuj31hc0u0gqo.jpg)

主机会根据 VNI 来区别不同的 vxlan 网络，不同的 vxlan 网络之间不会相互影响。如果再加上 network namespace，就能实现更复杂的网络结构。


## 参考资料

- [Vincent Bernat -VXLAN & Linux](https://vincent.bernat.im/en/blog/2017-vxlan-linux)
- [Linux Kernel Documentation: vxlan](https://www.kernel.org/doc/Documentation/networking/vxlan.txt)
- [Software Defined network using VXLAN](http://events.linuxfoundation.org/sites/events/files/slides/2013-linuxcon.pdf)
- [Docker Multi Host Networking: Overlay to the Rescue](https://www.singlestoneconsulting.com/-/media/files/docker-multi-host-networking-overlays-to-the-rescue.pdf?la=en)
- [Deep Dive into Docker Overlay Network Part 3](http://techblog.d2-si.eu/2017/08/20/deep-dive-into-docker-overlay-networks-part-3.html)