---
layout: post
title: "docker 跨主机网络：overlay 简介"
excerpt: "在多台机器上运行 docker 容器的时候，一会会碰到跨主机的容器通信问题。docker 目前已经有自带的解决方案来实现这个功能，这篇文章就讲讲 docker overlay 网络。"
categories: blog
tags: [docker, network, container, overlay]
comments: true
share: true
---

## 简介

docker 在早前的时候没有考虑跨主机的容器通信，这个特性直到 docker 1.9 才出现。在此之前，如果希望位于不同主机的容器能够通信，一般有几种方法：

- 使用端口映射：直接把容器的服务端口映射到主机上，主机直接通过映射出来的端口通信
- 把容器放到主机所在的网段：修改 docker 的 ip 分配网段和主机一致，还要修改主机的网络结构
- 第三方项目：flannel，weave 或者 pipework 等，这些方案一般都是通过 SDN 搭建 overlay 网络达到容器通信的

随着 docker 1.9 的发布，一个新的网络模型被开发出来（后面会写一篇文章专门介绍 docker 的网络项目 libnetwork）。除了能更方便地按照需求来搭建自己的网络方案，这次发布还让 docker 具备了跨主机通信的功能。

这篇文章介绍 docker swarm，和 docker overlay 网络（ docker 自带的跨主机网络模型），看看不同主机是怎么通信的。


使用 overlay 网络需要满足下面的这些条件：

- 正常工作的 key-value 存储服务，比如 consul、etcd、zookeeper 等
- 可以访问到 key-value 服务的主机集群
- 集群中每台机器都安装并运行 docker daemon
- 集群中每台机器的 hostname 都是唯一的，因为 key-value 服务是通过 hostname 标识每台主机的

## 安装 docker swarm 环境

注意： docker overlay 网络可以单独使用，不是必须和 swarm 绑定在一起的。这里使用 swarm，是因为它的简单易用，并且更容易说明问题。

先介绍一下 docker swarm， 这是 docker 开发的容器集群管理工具，和 docker API 兼容性很好，但目前功能不是很强大。

![](http://blog.daocloud.io/wp-content/uploads/2015/01/swarmarchitecture.jpg)

废话不多说，我们先来搭建一套 docker swarm 环境。这里的所有操作都是在我的机器上进行的，使用了 docker-machine 在 virtualbox 上面安装主机。docker-machine 提供了方便集成 swarm 的功能，所以安装起来并不复杂。

为了简化这个过程，我写了脚本来一键跑完这个过程（脚本我已经放到 [github 上](https://gist.github.com/cizixs/eee61a0ae65c6be30b74b80a9753efb3)）：

    #!/bin/bash

    set -e

    create_kv() {
        echo Creating kvstore machine.
        docker-machine create -d virtualbox \
            --engine-opt="registry-mirror=http://houchaohann.m.alauda.cn" \
            kvstore
        docker $(docker-machine config kvstore) run -d \
            -p "8500:8500" \
            progrium/consul --server -bootstrap-expect 1
    }

    create_master() {
        echo Creating cluster master
        kvip=$(docker-machine ip kvstore)

        docker-machine create -d virtualbox \
            --swarm --swarm-master \
            --swarm-discovery="consul://${kvip}:8500" \
            --engine-opt="cluster-store=consul://${kvip}:8500" \
            --engine-opt="cluster-advertise=eth1:2376" \
            --engine-opt="registry-mirror=http://houchaohann.m.alauda.cn" \
            swarm-manager
    }

    create_nodes(){
        kvip=$(docker-machine ip kvstore)
        echo Creating cluster nodes
        for i in 1 2; do
            docker-machine create -d virtualbox \
                --swarm \
                --swarm-discovery="consul://${kvip}:8500" \
                --engine-opt="cluster-store=consul://${kvip}:8500" \
                --engine-opt="cluster-advertise=eth1:2376" \
                --engine-opt="registry-mirror=http://houchaohann.m.alauda.cn" \
                swarm-node${i}
        done
    }


    teardown(){
        docker-machine rm kvstore -y
        docker-machine rm -y swarm-manager
        for i in 1 2; do
            docker-machine rm -y swarm-node${i}
        done
    }

    case $1 in
        up)
            create_kv
            create_master
            create_nodes
            ;;
        down)
            teardown
            ;;
        *)
            echo "Unknow command..."
            exit 1
            ;;
    esac

运行 `./cluster.sh up` 就能自动生成四台机器：

- 一台 kvstore运行 consul 服务
- 一台 swarm master 机器，运行 swarm manager 服务
- 两台 swarm node 机器，都是运行了 swarm node 服务和 docker daemon 服务

注意：上面的脚本设置了某国内厂家的 registry-mirror 来加速镜像的下载，你也可以根据自己的需求进行修改。

怎么验证集群已经正确安装呢？通过 client 和 swarm manager 交互，打印出来集群的状态就搞定了：

    ➜  eval $(docker-machine env --swarm swarm-manager)
    ➜  docker info
    Containers: 4
     Running: 4
     Paused: 0
     Stopped: 0
    Images: 3
    Server Version: swarm/1.2.3
    Role: primary
    Strategy: spread
    Filters: health, port, containerslots, dependency, affinity, constraint
    Nodes: 3
     swarm-manager: 192.168.99.136:2376
      └ ID: NHHY:6GRG:PVKL:BUIX:Z4TH:626A:BCTR:UFBM:BAP5:H4BJ:DUPO:UMJ2
      └ Status: Healthy
      └ Containers: 2
      └ Reserved CPUs: 0 / 1
      └ Reserved Memory: 0 B / 1.021 GiB
      └ Labels: executiondriver=, kernelversion=4.4.12-boot2docker, operatingsystem=Boot2Docker 1.11.2 (TCL 7.1); HEAD : a6645c3 - Wed Jun  1 22:59:51 UTC 2016, provider=virtualbox, storagedriver=aufs
      └ UpdatedAt: 2016-06-13T04:20:30Z
      └ ServerVersion: 1.11.2
     swarm-node1: 192.168.99.137:2376
      └ ID: O7QX:ZL3Y:WOCG:W4PP:2GDF:RCMM:K5PB:VSZE:GXE5:4M6C:JPHE:BWHM
      └ Status: Healthy
      └ Containers: 1
      └ Reserved CPUs: 0 / 1
      └ Reserved Memory: 0 B / 1.021 GiB
      └ Labels: executiondriver=, kernelversion=4.4.12-boot2docker, operatingsystem=Boot2Docker 1.11.2 (TCL 7.1); HEAD : a6645c3 - Wed Jun  1 22:59:51 UTC 2016, provider=virtualbox, storagedriver=aufs
      └ UpdatedAt: 2016-06-13T04:20:46Z
      └ ServerVersion: 1.11.2
     swarm-node2: 192.168.99.138:2376
      └ ID: RX4S:4UJK:CNCE:IG4V:LP7Y:ZQDL:VGZM:SXUJ:7INW:5PS7:RDLI:AK6A
      └ Status: Healthy
      └ Containers: 1
      └ Reserved CPUs: 0 / 1
      └ Reserved Memory: 0 B / 1.021 GiB
      └ Labels: executiondriver=, kernelversion=4.4.12-boot2docker, operatingsystem=Boot2Docker 1.11.2 (TCL 7.1); HEAD : a6645c3 - Wed Jun  1 22:59:51 UTC 2016, provider=virtualbox, storagedriver=aufs
      └ UpdatedAt: 2016-06-13T04:20:48Z
      └ ServerVersion: 1.11.2
    Plugins:
     Volume:
     Network:
    Kernel Version: 4.4.12-boot2docker
    Operating System: linux
    Architecture: amd64
    CPUs: 3
    Total Memory: 3.063 GiB
    Name: 729089ea0dca
    Docker Root Dir:
    Debug mode (client): false
    Debug mode (server): false
    WARNING: No kernel memory limit support

可以看到和单机的 `docker info` 不同的是：这里还打印出了集群的信息，以及集群中每台机器的信息。

注意：使用 `eval` 命令的时候多了 `--swarm` 参数，这样环境变量就会设置成和 swarm API 打交道啦。

## 创建 overlay 网络和容器

好了，环境准备 ok，正式开工吧！
下面创建 overlay network `multi`，然后创建两个容器放到这个网络，最后测试两个容器的连通性！

先创建 overlay 网络：

    ➜  docker network create -d overlay net1
    b29b16fae0516e5cde7d5a044b19fcbb62033ff1b4c3d4ba7a558e396bf47e5f
    ➜  docker network ls
    NETWORK ID          NAME                   DRIVER
    b29b16fae051        net1                   overlay
    edc4e05afb08        swarm-manager/bridge   bridge
    14298a4c6e37        swarm-manager/host     host
    c9ca0f7f09b4        swarm-manager/none     null
    95429bdaf5cf        swarm-node1/bridge     bridge
    641ede08038e        swarm-node1/host       host
    aaf1710f8f1b        swarm-node1/none       null
    9a12b0e2b2da        swarm-node2/bridge     bridge
    a9eafa21c06d        swarm-node2/host       host
    7a6015ebbc99        swarm-node2/none       null

`docker network` 命令原来管理容器的网络，第一个命令我们创建了一个名字叫 `net1` 的 overlay，第二个命令查看目前所有的网络。可以发现：

- 每台机器上已经有了 bridge、host、none 三种网络，对应于我们之前[讲过的容器网络模式](http://cizixs.com/2016/06/12/docker-network-modes-explained)
- overlay network 不属于任何一台主机，它属于整个集群

注：更多网络的命令可以参考 `docker network --help` 帮助文档。**为了防止网段冲突，可以使用 `--subnet` 参数指定创建的网段。**

简单起见，我们就创建两个 busybox 容器好了。

    ➜  docker run -d --net=net1 --name=c1 busybox top
    a7de0f1173f62518deb0364ec802133e15605bee5bc20b758cb734f668286b60
    ➜  docker ps
    CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
    a7de0f1173f6        busybox             "top"               8 seconds ago       Up 6 seconds                            swarm-node2/c1

只要使用 `--net` 指定网络名字，我们创建的容器就在对应的网络啦！`docker ps` 可以看到 `NAMES` 一栏，容器名字之前还有所在主机的名字。

为了保证第二个容器放到另外一台主机上，我们使用 docker swarm 提供的功能做到这一点。

    ➜  docker run -d --net=net1 --name=c2 -e constraint:node==swarm-node1 busybox top
    20b0c909cbf8e83782f8744cb62cbf2dc142098254c92d74ef30dbfaf3e0c677
    ➜  docker ps
    CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
    20b0c909cbf8        busybox             "top"               4 seconds ago       Up 3 seconds                            swarm-node1/c2
    a7de0f1173f6        busybox             "top"               7 minutes ago       Up 7 minutes                            swarm-node2/c1

注：更多关于 swarm 调度的内容可以参考[官方教程](https://docs.docker.com/swarm/scheduler/filter/)，这里就不多讲了。

看一下 `net1` 的详情：

    ➜  docker network inspect net1
    [
        {
            "Name": "net1",
            "Id": "b29b16fae0516e5cde7d5a044b19fcbb62033ff1b4c3d4ba7a558e396bf47e5f",
            "Scope": "global",
            "Driver": "overlay",
            "EnableIPv6": false,
            "IPAM": {
                "Driver": "default",
                "Options": {},
                "Config": [
                    {
                        "Subnet": "10.0.0.0/24",
                        "Gateway": "10.0.0.1/24"
                    }
                ]
            },
            "Internal": false,
            "Containers": {
                "20b0c909cbf8e83782f8744cb62cbf2dc142098254c92d74ef30dbfaf3e0c677": {
                    "Name": "c2",
                    "EndpointID": "22bf7a8621f4bc1ccdfd5c46d7514da88ab8f0a541da6e0851b6afe4ed3b49ac",
                    "MacAddress": "02:42:0a:00:00:03",
                    "IPv4Address": "10.0.0.3/24",
                    "IPv6Address": ""
                },
                "a7de0f1173f62518deb0364ec802133e15605bee5bc20b758cb734f668286b60": {
                    "Name": "c1",
                    "EndpointID": "ccfe9fdb12389c1bada0d4473be16e84d20aff0fef9ae7f86fcfc21e218c4e3e",
                    "MacAddress": "02:42:0a:00:00:02",
                    "IPv4Address": "10.0.0.2/24",
                    "IPv6Address": ""
                }
            },
            "Options": {},
            "Labels": {}
        }
    ]

可以看到 overlay 的基本信息，还有我们刚刚创建容器的网络信息也在里面。下面就测试一下两个容器能否互相 ping 通：

    ➜  docker exec c1 ping -c 3 10.0.0.3
    PING 10.0.0.3 (10.0.0.3): 56 data bytes
    64 bytes from 10.0.0.3: seq=0 ttl=64 time=0.476 ms
    64 bytes from 10.0.0.3: seq=1 ttl=64 time=0.484 ms
    64 bytes from 10.0.0.3: seq=2 ttl=64 time=0.615 ms

    --- 10.0.0.3 ping statistics ---
    3 packets transmitted, 3 packets received, 0% packet loss
    round-trip min/avg/max = 0.476/0.525/0.615 ms

    ➜  docker exec c2 ping -c 3 10.0.0.2
    PING 10.0.0.2 (10.0.0.2): 56 data bytes
    64 bytes from 10.0.0.2: seq=0 ttl=64 time=0.572 ms
    64 bytes from 10.0.0.2: seq=1 ttl=64 time=0.745 ms
    64 bytes from 10.0.0.2: seq=2 ttl=64 time=0.626 ms

    --- 10.0.0.2 ping statistics ---
    3 packets transmitted, 3 packets received, 0% packet loss
    round-trip min/avg/max = 0.572/0.647/0.745 ms

    ➜  docker exec c2 ping -c 3 c1
    PING c1 (10.0.0.2): 56 data bytes
    64 bytes from 10.0.0.2: seq=0 ttl=64 time=1.075 ms
    64 bytes from 10.0.0.2: seq=1 ttl=64 time=0.506 ms
    64 bytes from 10.0.0.2: seq=2 ttl=64 time=0.502 ms

    --- c1 ping statistics ---
    3 packets transmitted, 3 packets received, 0% packet loss
    round-trip min/avg/max = 0.502/0.694/1.075 ms

注意：在最后一个命令中，直接使用容器的名字也能 ping 通。

实验就此完成，我们已经看到即使在两台不同的主机上，在同一个 overlay 网络中的容器也是联通的。你可以自己多创建几个 overlay 网络，多创建几个更有用的容器试一下。

那么，最后一个部分就讲讲 docker 是怎么实现 overlay 网络的通信的！

## overlay 网络模型分析

先进入到容器里看一下网络情况：

    ➜  docker exec c1 ip addr
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
           valid_lft forever preferred_lft forever
    10: eth0@if11: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue
        link/ether 02:42:0a:00:00:02 brd ff:ff:ff:ff:ff:ff
        inet 10.0.0.2/24 scope global eth0
           valid_lft forever preferred_lft forever
        inet6 fe80::42:aff:fe00:2/64 scope link
           valid_lft forever preferred_lft forever
    13: eth1@if14: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
        link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff
        inet 172.18.0.2/16 scope global eth1
           valid_lft forever preferred_lft forever
        inet6 fe80::42:acff:fe12:2/64 scope link
           valid_lft forever preferred_lft forever

发现容器有两个网口 `eth0` 和 `eth1`，其中 `eth0` 是我们在 `docker network inspect` 中看到的，它是 veth pair 中的一个，对应着 `if11` 网络端口；另外一个属于 `172.18.0.1/16` 网段，并不是 `docker0` 所在的 `172.17.0.1/16`，它对应的 veth pair 是 `if14`。interesting！这个疑问我们先不要管，继续看网络的路由，发现两个网段也都有自己的路由规则：

    ➜  docker exec c1 ip route
    default via 172.18.0.1 dev eth1
    10.0.0.0/24 dev eth0  src 10.0.0.2
    172.18.0.0/16 dev eth1  src 172.18.0.2


除了多出来一个网段，并没有看到什么奇怪的东西。那么，主机上的情况呢？

    docker@swarm-node2:~$ ip addr

    3: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:d8:58:ef brd ff:ff:ff:ff:ff:ff
        inet 10.0.2.15/24 brd 10.0.2.255 scope global eth0
           valid_lft forever preferred_lft forever
        inet6 fe80::a00:27ff:fed8:58ef/64 scope link
           valid_lft forever preferred_lft forever
    4: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:3f:12:de brd ff:ff:ff:ff:ff:ff
        inet 192.168.99.138/24 brd 192.168.99.255 scope global eth1
           valid_lft forever preferred_lft forever
        inet6 fe80::a00:27ff:fe3f:12de/64 scope link
           valid_lft forever preferred_lft forever
    5: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
        link/ether 02:42:a7:36:e5:66 brd ff:ff:ff:ff:ff:ff
        inet 172.17.0.1/16 scope global docker0
           valid_lft forever preferred_lft forever
        inet6 fe80::42:a7ff:fe36:e566/64 scope link
           valid_lft forever preferred_lft forever
    7: veth0a68563@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
        link/ether 9a:39:cb:60:0f:29 brd ff:ff:ff:ff:ff:ff
        inet6 fe80::9839:cbff:fe60:f29/64 scope link
           valid_lft forever preferred_lft forever
    12: docker_gwbridge: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
        link/ether 02:42:e4:b2:28:24 brd ff:ff:ff:ff:ff:ff
        inet 172.18.0.1/16 scope global docker_gwbridge
           valid_lft forever preferred_lft forever
        inet6 fe80::42:e4ff:feb2:2824/64 scope link
           valid_lft forever preferred_lft forever
    14: veth3fcaaef@if13: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker_gwbridge state UP group default
        link/ether be:a1:f9:a3:a4:3e brd ff:ff:ff:ff:ff:ff
        inet6 fe80::bca1:f9ff:fea3:a43e/64 scope link
           valid_lft forever preferred_lft forever

除了 docker0 之外，还多了 `docker_gwbridge` 这个网口。并且找到了 `if14` 这个端口，它对应的 `if13` 就是容器里的 `eth1`。而且 `if13` 对应的网段就是 `docker_gwbridge` 所在的网段，
使用  `brctl` 命令也发现 veth 网口是绑定到 `docker_gwbridge`，而不是 `docker0` 的。

现在搞明白了一件事：容器中 `eth1` 是连接到新创立的 `docker_gwbridge` 虚拟网桥上，它的作用和之前 docker0 一样，专门做 overlay 网络中的通主机上容器的通信、容器和外部的通信工作。问题是：容器的 `eth0`，也就是 overlay 网络为什么看不到信息呢？

自然，我们就想到它们一定是在独立的 network namespace，被隐藏了起来。为了方便，我们先把它们找出来，连接到 ip netns 能管理的地方：

    sudo ln -s /var/run/docker/netns /var/run/netns

然后，执行 `ip netns ls` 就能看到所有在 netns：

    root@swarm-node2:/home/docker# ip netns ls
    24aba2d4f90a
    1-b29b16fae0
    8882bdcea169

哎！我们发现了三个 namespace：一个容器 `c1`，一个属于容器 `swarm agent`，那么另外一个就属于 `overlay` 啦！而且很容器猜想那个名称中有 `-` 符号的很可能是 overlay 网络创建的 namespace：

    root@swarm-node2:/home/docker# ip netns exec 1-b29b16fae0 ip addr
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
           valid_lft forever preferred_lft forever
    2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
        link/ether 6e:b8:1f:82:13:63 brd ff:ff:ff:ff:ff:ff
        inet 10.0.0.1/24 scope global br0
           valid_lft forever preferred_lft forever
        inet6 fe80::68e0:f0ff:fe19:e88c/64 scope link
           valid_lft forever preferred_lft forever
    9: vxlan1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UNKNOWN group default
        link/ether 6e:b8:1f:82:13:63 brd ff:ff:ff:ff:ff:ff
        inet6 fe80::6cb8:1fff:fe82:1363/64 scope link
           valid_lft forever preferred_lft forever
    11: veth2@if10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP group default
        link/ether d2:83:53:55:8c:98 brd ff:ff:ff:ff:ff:ff
        inet6 fe80::d083:53ff:fe55:8c98/64 scope link
           valid_lft forever preferred_lft forever

果然！我们在这里找到了消失的 `if11`，之外，还有两个重要的发现：`br0` 和 `vxlan1`。通过名字和网段，我们猜测 `br0` 是这里的虚拟网桥，那么 `vxlan1` 虽然不知道具体做什么的，但应该和 `VxLAN` 有关。

这个 namespace 的路由规则很简单，都是发送到 `br0` 的。

    root@swarm-node2:/home/docker# ip netns exec 1-b29b16fae0 ip route
    10.0.0.0/24 dev br0  proto kernel  scope link  src 10.0.0.1

我们继续看 `vxlan1` 这个东西，使用 `ip -d link` 命令查看它的类型：

    root@swarm-node2:/home/docker# ip netns exec 1-b29b16fae0 ip -d link
    2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT group default
        link/ether 6e:b8:1f:82:13:63 brd ff:ff:ff:ff:ff:ff promiscuity 0
        bridge
    9: vxlan1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UNKNOWN mode DEFAULT group default
        link/ether 6e:b8:1f:82:13:63 brd ff:ff:ff:ff:ff:ff promiscuity 1
        vxlan id 256 srcport 0 0 dstport 4789 proxy l2miss l3miss ageing 300
        bridge_slave
    11: veth2@if10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP mode DEFAULT group default
        link/ether d2:83:53:55:8c:98 brd ff:ff:ff:ff:ff:ff promiscuity 1
        veth
        bridge_slave

发现它是  `vxlan` 类型的，并且和 `veth2` 一样是 bridge 的 salve（也就是连到虚拟网桥的）。这里就需要了解一点 vxlan 的知识了：这里的 `vxlan1` 是一个 `VTEP`（全称是 VXLAN Tunnel End-Point），VxLAN 的隧道端点，它是 VxLAN 中重要的部分，所有数据报文的校验、封装和转发都是在这里进行的。

注：VxLAN 是一个复杂的概念，这里只需要理解所有的数据报文都是在这里转发，发送到主机的网络就行了。

下面看看 c1（10.0.0.2） 发送的 ping 报文是怎么发送到 c2(10.0.0.3) 的：

1. c1 找到路由发现目的 ip 可以直达，于是发送 arp 报文找到目标的 mac 地址，封包，通过 eth0 发送出去
2. 报文传输到 veth pair 的另外一端 veth2，并发送到其绑定的虚拟交换机 br0
3. br0 会将报文转交给 vxlan1，这里可以参考 arp 地址来确定这一点：

        root@swarm-node2:/home/docker# ip netns exec 1-b29b16fae0 ip neigh
        10.0.0.3 dev vxlan1 lladdr 02:42:0a:00:00:03 PERMANENT

4. vxlan 会查询 consul 中保存的目的主机地址，完成报文的封装并通过主机地址 eth1 转发出去
5. 通过中间网络和路由，报文被发送到目的主机
6. 目的主机介绍到报文，发现是 VxLAN 报文，把它转交给 vxlan 设备，也就是 `vxlan1` 处理
7. `vxlan1` 解包，取出里面被封装的报文，把它转交给 `br0`
8. `br0` 发现本文是发送到连到它上面的某个容器的，将报文交给容器

## 参考资料

- [理解Docker跨多主机容器网络
](http://tonybai.com/2016/02/15/understanding-docker-multi-host-networking/)
- [利用虚拟网桥实现Docker容器的跨主机访问](http://www.cnblogs.com/ruiqingzhang/p/4463971.html)
- [Docker Network Reborn](http://container42.com/2015/10/30/docker-networking-reborn/)
- [深入浅出 docker swarm](http://blog.daocloud.io/swarm_analysis_part1/)
- [docker 官方一篇关于 overlay 网络的教程](https://docs.docker.com/engine/userguide/networking/get-started-overlay/)
