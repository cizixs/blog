---
layout: post
title: "flannel 网络模型"
excerpt: "flannel 是 CoreOS 公司开发的网络项目，旨在解决 docker 跨主机的容器通信问题，目前主要被用在 kubernetes 中。"
categories: blog
tags: [docker, network, container, overlay, flannel]
comments: true
share: true
---


## 简介

[Flannel](https://github.com/coreos/flannel) 是CoreOS 下面的一个项目，目前被使用在 kubernetes 中，用于解决 docker 容器直接跨主机的通信问题。它的主要思路是：**预先留出一个网段，每个主机使用其中一部分，然后每个容器被分配不同的 ip；让所有的容器认为大家在同一个直连的网络，底层通过 UDP/VxLAN 等进行报文的封装和转发。**这么说还是会很不清不楚，那么这篇文章就试图解释到底 flannel 是怎么回事，希望读完本文再来看这段话能够明白它的意思。


## 安装和配置

### 启动主机

docker 要运行的主机可以是物理机，也可以是本地的虚拟机，或者公有云上的机器。根据自己的情况创建至少两台机器作为容器运行的主机，我这里就直接使用 docker 提供的 toolbox 在本地运行：

    docker-machine create -d vitrualbox kvstore
    docker-machine create -d virtualbox node1
    docker-machine create -d virtualbox node2

因为 flannel 要使用 etcd 服务，我这里也创建了一台机器来单独运行这个服务。如果你已经有了 etcd server，可以不用创建，直接使用就行。

### 安装 docker

在每一个要运行容器的节点上安装 docker（这里使用 daocloud 提供的脚本来安装，你也可以参考[官网的安装说明](https://docs.docker.com/engine/installation/)）：

    curl -sSL https://get.daocloud.io/docker | sh

### 安装 etcd server

flannel 默认使用 etcd 作为网络配置的存储，以便每台主机知道整个集群的网络情况。所以我们要先搭建 etcd server（因为 etcd 不是这篇文章的重点，所以不做重点介绍，只是简单地启动单点的服务）。

    docker run --rm -it -p 4001:4001 -p 7001:7001 -v /var/etcd/:/data microbox/etcd:latest -name etcd

### 配置 flannel

配置 flannel 可以使用的网段（每个主机上被分配的网段都是从这个网段中动态获取的），**注意不要和你网络中已经使用的网络冲突**：

    curl -X PUT http://$(dm ip default):4001/v2/keys/coreos.com/network/config -d value='{"Network": "172.17.0.1/16"}'

这个命令只需要操作一次就行，当然你可以使用 etcdctl 或者其他工具来达到相同的效果。这里就不多解释了！

然后，在每个节点上启动 flannel，指定需要的参数（**使用 `iface` 指定要作为要使用的网络接口，不同主机一定有不同的 ip**。我这里因为是通过 docker-machine 在 virtualbox 启动的节点，eth0 是默认的 NAT 接口，不能直接用来通信，所以指定 eth1）：

    sudo ./flanneld -etcd-endpoints="http://192.168.99.100:4001" -iface eth1

最后，修改 docker 的配置文件：

    source /run/flannel/subnet.env
    sudo rm /var/run/docker.pid
    sudo ifconfig docker0 ${FLANNEL_SUBNET}
    sudo docker daemon --bip=${FLANNEL_SUBNET} --mtu=${FLANNEL_MTU}

使得 docker 能够使用 flannel 的网络配置。

这里要解释一下：启动 flannel 程序之后，会根据 etcd 中的信息，自动分配所在主机的网络，然后生成 `/run/flannel/subnet.env` 文件，主要是 `FLANNEL_SUBNET` 和 `FLANNEL_MTU` 两个变量；然后我们修改 docker0 的 ip 地址到刚分配的网段的默认 ip；最后配置 docker 可以分配的网段为刚分配的网段，并修改 mtu 的值。

## 网络配置信息

### etcd 的配置内容

默认情况下，flannel 相关的信息存放在 `/coreos.com/network` 下面（当然你也可以在启动 flannel 的时候修改这个路径）：

    ➜  http http://192.168.99.100:4001/v2/keys/coreos.com/network/
    HTTP/1.1 200 OK
    Content-Type: application/json
    Date: Wed, 08 Jun 2016 03:50:00 GMT
    Transfer-Encoding: chunked
    X-Etcd-Index: 14
    X-Raft-Index: 26858
    X-Raft-Term: 0

    {
        "action": "get",
        "node": {
            "createdIndex": 3,
            "dir": true,
            "key": "/coreos.com/network",
            "modifiedIndex": 3,
            "nodes": [
                {
                    "createdIndex": 3,
                    "key": "/coreos.com/network/config",
                    "modifiedIndex": 3,
                    "value": "{\"Network\": \"172.17.0.1/16\"}"
                },
                {
                    "createdIndex": 4,
                    "dir": true,
                    "key": "/coreos.com/network/subnets",
                    "modifiedIndex": 4
                }
            ]
        }
    }

可以看到除了之前配置的 `/coreos.com/network/config` 网段，还增加了 `/coreos.com/network/subnets` 的目录（里面存放着每台主机的网络信息）。继续看里面的内容：

    ➜ http http://192.168.99.100:4001/v2/keys/coreos.com/network/subnets/
    HTTP/1.1 200 OK
    Content-Type: application/json
    Date: Wed, 08 Jun 2016 04:08:51 GMT
    Transfer-Encoding: chunked
    X-Etcd-Index: 16
    X-Raft-Index: 29122
    X-Raft-Term: 0

    {
        "action": "get",
        "node": {
            "createdIndex": 4,
            "dir": true,
            "key": "/coreos.com/network/subnets",
            "modifiedIndex": 4,
            "nodes": [
                {
                    "createdIndex": 15,
                    "expiration": "2016-06-09T04:04:48.891438435Z",
                    "key": "/coreos.com/network/subnets/172.17.80.0-24",
                    "modifiedIndex": 15,
                    "ttl": 86157,
                    "value": "{\"PublicIP\":\"192.168.99.130\"}"
                },
                {
                    "createdIndex": 16,
                    "expiration": "2016-06-09T04:08:43.714803058Z",
                    "key": "/coreos.com/network/subnets/172.17.76.0-24",
                    "modifiedIndex": 16,
                    "ttl": 86392,
                    "value": "{\"PublicIP\":\"192.168.99.131\"}"
                }
            ]
        }
    }

里面有两个主机分配的网络信息，注意 `PublicIP` 就是我们指定的 `iface=eth1` 的地址，用来唯一标识一台主机。每次 flannel 启动的时候都会来这里获取所在主机的网络信息：如果发现对应的 ip 已经有对应的网络记录，就直接使用；如果发现没有，就从可用的网段里分配出来一个，并写到这里。

### 主机上的配置

在主机上，flannel 做了几个改动：增加了 flannel0 网口，修改了 docker0 的网络信息。那么对应的路由信息也会发生改变：

    root@node1:/home/docker/flannel-0.5.5# ip route
    default via 10.0.2.2 dev eth0  metric 1
    10.0.2.0/24 dev eth0  proto kernel  scope link  src 10.0.2.15
    172.17.0.0/16 dev flannel0  proto kernel  scope link  src 172.17.80.0
    172.17.80.0/24 dev docker0  proto kernel  scope link  src 172.17.80.1

可以看到除了 docker0 相关的项之外，还有 flannel0 的项。如果报文是发送出去的，那么会走 `flannel0` 出去，如果报文是进来的，那么会通过 `docker0` 进入到容器。

## 架构介绍

flannel 默认使用 8285 端口作为 UDP 封装报文的端口，VxLan 的话使用 8472 端口。

![](http://dockerone.com/uploads/article/20150826/5bf473d89214a5d1e84f67ad231dd263.png)

那么一条网络报文是怎么从一个容器发送到另外一个容器的呢？

1. 容器直接使用目标容器的 ip 访问，默认通过容器内部的 eth0 发送出去
2. 报文通过 veth pair 被发送到 vethXXX
3. vethXXX 是直接连接到虚拟交换机 docker0 的，报文通过虚拟 bridge docker0 发送出去
4. 查找路由表，外部容器 ip 的报文都会转发到 flannel0 虚拟网卡，这是一个 P2P 的虚拟网卡（关于这一点的工作原理，我也不是很清楚），然后报文就被转发到监听在另一端的 flanneld
5. flanneld 通过 etcd 维护了各个节点之间的路由表，把原来的报文 UDP 封装一层，通过配置的 `iface` 发送出去
6. 报文通过主机之间的网络找到目标主机
7. 报文继续往上，到传输层，交给监听在 8285 端口的 flanneld 程序处理
7. 数据被解包，然后发送给 flannel0 虚拟网卡
8. 查找路由表，发现对应容器的报文要交给 docker0
9. docker0 找到连到自己的容器，把报文发送过去

## 报文分析

最后，我们来抓包看看网络的报文。我们使用了 [容器化的 tcpdump](https://hub.docker.com/r/corfr/tcpdump/) 工具在主机上抓包：

    docker run --net=host -v $PWD:/data corfr/tcpdump -i any -w /data/flannel0.pcap

然后在 wireshark 中看一下封装的报文：

![](http://ww2.sinaimg.cn/large/728b3d6dgw1f4nvqeigjqj21ix0wh19x.jpg)

这里使用了 wireshark 的 `decode as` 功能把被封装的报文显示出来。可以看到主机间是在 UDP 8285 端口通信的，报文中包含了容器间真正的网络报文，比如这里的 ping 包（ICMP 协议报文）。

## 需要注意的事项

- flannel 默认采用 UDP 封装报文，在高并发情况下会有丢包问题
- 因为封装报文是在用户区进行的，会有一定的性能损失
- 要求所有主机在同一个网络，可以直接路由
- 会导致 ip 漂移：删除一台容器重新部署，容器的 ip 很可能会发生变化（新部署的容器落在另外一台主机上一定会导致 ip 不同）

## 参考资料

- [一篇文章带你了解 Flannel](http://dockone.io/article/618)
- [weave 和 flannel 性能测试](http://www.generictestdomain.net/docker/weave/networking/stupidity/2015/04/05/weave-is-kinda-slow/)
- [Battlefield: Calico, Flannel, Weave and Docker Overlay Network
](http://chunqi.li/2015/11/15/Battlefield-Calico-Flannel-Weave-and-Docker-Overlay-Network/)
