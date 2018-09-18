---
layout: post
title: "weave 网络模型"
excerpt: "weave 是一家解决容器网络问题的项目，它提供了很多使用的功能，和 docker 集成得也很不错。"
categories: blog
tags: [docker, network, container, overlay, weave]
comments: true
share: true
---


## 简介

weave 是 weaveworks 公司核心项目，也是解决 docker 网络问题的。和其他项目比较，weave 提供的功能更多，和其他系统的集成更完善。

weave 不会修改 docker 也有的网络配置，而是添加在容器中添加额外的虚拟网卡来搭建自己的网络。

## 安装

老规矩，下面所有的环境都是在我 mac 笔记本上通过 docker-machine 创建和运行的。

这次我们就创建两台虚拟主机，主机名分别为 `weave-1` 和 `weave-2`。请确保你安装的 docker 版本不小于 1.11，并且熟悉 `curl` 等命令的操作。

weave 不需要集中式的 key-value 存储，所以安装和运行都很简单。直接把 weave 二进制文件下载到系统中就可以了，可以参考官网的命令：

    sudo curl -L git.io/weave -o /usr/local/bin/weave
    sudo chmod a+x /usr/local/bin/weave

运行命令也比较简单，直接运行 `weave launch`，weave 就会下载需要的容器就运行，等到命令运行结束，网络就配置好了。

那么既然没有集中式存储， weave 是怎么建立集群，知道其他主机信息的呢？说起来比较简单，在运行 `weave launch` 的时候，后面跟上主机的 ip 或者 hostname，这两台机器就会自动建立集群，并同步所有需要的信息。

在第一台机器上，我们直接运行 `weave lacunch` ，weave 会下载并运行三个 docker 容器。在我运行的环境里，它们的版本都是 `1.5.2`：

    docker@weave-1:~$ docker ps
    CONTAINER ID        IMAGE                        COMMAND                  CREATED              STATUS              PORTS               NAMES
    dcae08dd6efc        weaveworks/plugin:1.5.2      "/home/weave/plugin"     50 seconds ago       Up 49 seconds                           weaveplugin
    c9c5d777719e        weaveworks/weaveexec:1.5.2   "/home/weave/weavepro"   About a minute ago   Up About a minute                       weaveproxy
    5252d8e8d581        weaveworks/weave:1.5.2       "/home/weave/weaver -"   About a minute ago   Up About a minute                       weave

在第二台机器上运行的命令稍有不同，需要把第一台机器的 ip 作为参数添加上去：

    docker@weave-2:~$ weave launch 192.168.99.156

同样，当命令运行完，机器上会运行三台 weave 有关的容器。运行 `weave status`，可以查看 weave 的状态信息：

    docker@weave-2:~$ weave status

            Version: 1.5.2 (up to date; next check at 2016/06/17 11:57:58)

            Service: router
           Protocol: weave 1..2
               Name: 9a:d4:60:be:bb:f4(weave-2)
         Encryption: disabled
      PeerDiscovery: enabled
            Targets: 1
        Connections: 1 (1 established)
              Peers: 2 (with 2 established connections)
     TrustedSubnets: none

            Service: ipam
             Status: idle
              Range: 10.32.0.0-10.47.255.255
      DefaultSubnet: 10.32.0.0/12

            Service: dns
             Domain: weave.local.
           Upstream: 10.0.2.3
                TTL: 1
            Entries: 0

            Service: proxy
            Address: unix:///var/run/weave/weave.sock

            Service: plugin
         DriverName: weave

更多关于 weave 的时候，可以参考 `weave --help` 的文档。

## 使用 weave

weave 有三种方式和 docker 进行集成，以便运行的容器跑在 weave 网络中。

1. 使用 `weave run` 命令直接运行容器
2. 使用 weave proxy，通过修改 `DOKCER_HOST` 环境变量的值，使 docker client 和 weave proxy 交互，weave proxy 和 docker daemon 交互，自动为容器配置网络，对用户透明
3. 使用 weave plugin，在运行容器的时候使用 `--net=weave` 参数

下面的教程直接使用第一种方法，其他两种方法的使用请自行参阅官方文档。

我们在 weave-1 主机上运行一台容器，按照老规矩命名为 `c1`：

    weave run -d --name c1 busybox top

同样，在 weave-2 主机上运行另外一台容器，名字为 `c2`：

    weave run -d --name c2 busybox top

从 c2 上 ping c1:

    root@weave-2:/home/docker# docker exec c2 ping -c 3 c1
    PING c1 (10.32.0.1): 56 data bytes
    64 bytes from 10.32.0.1: seq=0 ttl=64 time=2.099 ms
    64 bytes from 10.32.0.1: seq=1 ttl=64 time=0.597 ms
    64 bytes from 10.32.0.1: seq=2 ttl=64 time=0.540 ms

    --- c1 ping statistics ---
    3 packets transmitted, 3 packets received, 0% packet loss
    round-trip min/avg/max = 0.540/1.078/2.099 ms

发现网络是联通的，说明 weave 正确配置，并实现了跨主机的容器通信。

## ip 分配

这一部分讲讲 weave 的 ip 分配机制，已经怎么配置分配机器来满足不同的需求。

weave 会保证容器 ip 都是唯一的，并且自动负责容器 ip 的分配和回收，你不需要配置任何东西就能得到这个好处。但是如果对分配有特定需求，还是要看一下分配原理的。

weave 默认使用的网段是 `10.32.0.0/12`，也就是从 `10.32.0.0` 到 `10.47.255.255`。如果这个网段已经被占用，weave 就会报错，这个时候必须使用 `--ipalloc-range` 参数手动指定 weave 要使用的网段。默认情况下，所有的容器都在同一个子网，不过可以通过 `--ipalloc-default-subnet` 指定分配的子网（这个子网必须在可分配网段里）。

那么 weave 的这些数据是保存在哪里的呢？目前每台机器上的分配数据是放在名为 `weavedb` 的容器上的，这个一个 data volume 容器，只负责数据的持久化。

### 控制 ip 的值

在有些时候，需要指定容器的 ip。在运行容器的时候，可以通过设置环境变量做到这一点，比如：

    docker run -e WEAVE_CIDR=10.2.1.2/24 -d awesome_container

就设置了容器的 ip 是 `10.2.1.2`，掩码是 `255.255.255.0`。

### 网络隔离

有些情况下，我们希望能做到容器网络的隔离，而不是集群中所有容器都能互相访问。weave 可以使用子网实现这个功能，前面已经提到怎么设置默认子网以及容器启动的时候指定子网。同一个子网的容器互相连通，而不同的子网中容器是隔离的。

当然，一个容器可以在多个子网中，通过和多个网络里的容器互连 。

    host1$ docker run -e WEAVE_CIDR="net:default net:10.2.2.0/24" -ti weaveworks/ubuntu

注：docker 允许同一个机器上的容器互通，为了完全隔离，需要在 docker daemon 启动参数添加上 `-icc=false`。

### 动态分配网络

如果在运行容器的时候还不能确定网络，可以使用 `-e WEAVE_CIDR=none` 参数先不分配网络信息；在后面使用  `weave attach ${container_id}` 给运行中的容器分配网络。相对应的，你也可以把容器从某个网络中移除，命令是 `weave detach net:10.2.2.0/24 ${container_id}`。

## 性能测试

注意：这只是非常简陋的性能测试实验，非常多的因素没有考虑也没有进行控制，只能给出一个大约的性能印象，不可作为生产环境的选择标准。

所有的测试都是直接运行 `mustaf
aakin/alpine-iperf` 容器的 `iperf` 命令，`iperf` 的使用方法不在这里给出解释，默认读者理解它的原理和用法（如果不理解，请自行搜索其用法）。

作为基准，我们先测试两台主机（virtualbox 虚拟机）之间的网络速度。
在一台机器上运行 `iperf` 的服务端模式：

    docker run --name iserver -d --net=host mustafaakin/alpine-iperf iperf -s

然后在另外一台机器运行 `iperf` 的客户端，测试两个应用之间的网络速度：

    root@weave-2:/home/docker# docker run -it --net=host mustafaakin/alpin
    e-iperf sh
    / # iperf -c 192.168.99.156
    ------------------------------------------------------------
    Client connecting to 192.168.99.156, TCP port 5001
    TCP window size: 85.0 KByte (default)
    ------------------------------------------------------------
    [  3] local 192.168.99.157 port 32776 connected with 192.168.99.156 port 5001
    [ ID] Interval       Transfer     Bandwidth
    [  3]  0.0-10.1 sec  1.62 GBytes  1.38 Gbits/sec

可以看到主机间的大概网速是 `1.38 Gbits/sec`。

如果把 iperf 服务端和客户端都放到 weave 的网络中运行，

    / # iperf -c 10.32.0.3
    ------------------------------------------------------------
    Client connecting to 10.32.0.3, TCP port 5001
    TCP window size: 45.0 KByte (default)
    ------------------------------------------------------------
    [  3] local 10.40.0.2 port 40822 connected with 10.32.0.3 port 5001
    [ ID] Interval       Transfer     Bandwidth
    [  3]  0.0-10.0 sec  1.48 GBytes  1.27 Gbits/sec

网速会稍微有点下降，也就是说 weave 跨主机模式会造成网络的损耗。

## 参考资料

- [weave 官方教程](https://www.weave.works/docs/)
- [weave: Network Management for Docker](https://xelatex.github.io/2015/11/14/Weave-Network-Management-for-Docker/)
