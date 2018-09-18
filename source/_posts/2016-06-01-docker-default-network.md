---
layout: post
title: "docker 容器默认的网络模型"
excerpt: "启动容器之后，默认就能从容器内部访问外网。这篇文章就介绍 docker 默认的网络是怎么配置的，使用了哪些技术。"
categories: blog
tags: [docker, mac, virtualbox, network, iptables]
comments: true
share: true
---

## 简介

这篇文章介绍我们启动容器的时候，默认情况下 docker 是怎么组织网络，保证容器是可以连通的。

这里将不再介绍 docker 的基础使用，而是默认读者已经熟悉了 docker 各种命令行和概念，能创建、运行、查看容器。

除此之外，还需要有一定的网络基础知识，包括但是不限于：

- 二层网络和三层网络的区别和作用
- ip 命令的使用
- linux network namespace 的概念
- virtual bridge 的概念
- veth pair 的功能

好，下面就让我们开始吧！

## docker 容器网络

在默认情况下，docker 会在 host 机器上新创建一个 `docker0` 的 bridge：可以把它想象成一个虚拟的交换机，所有的容器都是连到这台交换机上面的。docker 会从私有网络中选择一段地址来管理容器，比如 `172.17.0.1/16`，这个地址根据你之前的网络情况而有所不同。

![](http://developerblog.info/content/images/2015/11/docker-turtles-communication.jpg)

> When Docker starts, it creates a virtual interface named docker0 on the host machine. It randomly chooses an address and subnet from the private range defined by RFC 1918 that are not in use on the host machine, and assigns it to docker0. Docker made the choice 172.17.42.1/16 when I started it a few minutes ago, for example — a 16-bit netmask providing 65,534 addresses for the host machine and its containers. The MAC address is generated using the IP address allocated to the container to avoid ARP collisions, using a range from 02:42:ac:11:00:00 to 02:42:ac:11:ff:ff.
-- Docker Official Document

通过 `ip addr` 命令可以查看主机上面所有的网络接口：

    root@swarm-node1:/home/docker# ip addr
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
           valid_lft forever preferred_lft forever
    3: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:f5:13:b4 brd ff:ff:ff:ff:ff:ff
        inet 10.0.2.15/24 brd 10.0.2.255 scope global eth0
           valid_lft forever preferred_lft forever
        inet6 fe80::a00:27ff:fef5:13b4/64 scope link
           valid_lft forever preferred_lft forever
    6: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
        link/ether 02:42:26:fd:f2:1f brd ff:ff:ff:ff:ff:ff
        inet 172.17.0.1/16 scope global docker0
           valid_lft forever preferred_lft forever
        inet6 fe80::42:26ff:fefd:f21f/64 scope link
           valid_lft forever preferred_lft forever

 `docker0` 就是所有魔法的关键，粗糙地说，是它连接了容器和主机网络。下面的部分会一步步解开它的面纱！

### 容器是怎么连接到外面的网络的？

![](http://www.linuxjournal.com/files/linuxjournal.com/ufiles/imagecache/large-550px-centered/u1002061/11833f1.png)

启动一个容器，进到里面的 shell，可以发现：默认情况下，容器内部能够访问外网（当然你本身机器要联通外网）。这个是怎么做到的呢？

每创建一个容器，docker 会新建一对 interfaces，这对 interfaces 最大的特性就是：从一个地方进去的网络报文都能在另外一个接口被接受，就像水管的两头。一个接口命名为 `eth0`，分配给容器内部；另外一个接口命名是 `veth**` 这样的形式，显示在 host 机器上，连接到 `docker0`。

**总结一下就是：docker 会在机器上自己维护一个网络，并通过 `docker0` 这个虚拟交换机和主机本身的网络连接在一起。**

我们运行一个 busybox 容器：

    ➜  ~ docker run -d busybox top
    991022faafc3af764e4f5bd2ba159722661f5b1599fb9d45b3d791b9e9cacf18

    ➜  ~ sudo brctl show
    bridge name	bridge id		STP enabled	interfaces
    docker0		8000.024250c829eb	no		vethde0e617

通过 `brctl` 命令（管理虚拟网桥的命令，可以使用 `apt-get install bridge-utils` 安装）可以看到 `docker0` 上新连接了一个 interface `vethde0e617`，然后查看它的详细信息：

    ➜  ~ ip addr show vethde0e617
    26: vethde0e617@if25: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
        link/ether aa:5f:5a:2c:19:d5 brd ff:ff:ff:ff:ff:ff link-netnsid 0
        inet6 fe80::a85f:5aff:fe2c:19d5/64 scope link
           valid_lft forever preferred_lft forever

它有 mac 地址（因为虚拟交换机需要 mac 地址才能转发报文），但并没有 ip 地址，只是被连接到 `docker0` 而已，ip 地址只存在于容器里面。

进入到容器内部，看一下另外一个 interface：

    ➜  ~ docker exec -it 5e03 sh
    / # ip addr
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
           valid_lft forever preferred_lft forever
    25: eth0@if26: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
        link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
        inet 172.17.0.2/16 scope global eth0
           valid_lft forever preferred_lft forever
        inet6 fe80::42:acff:fe11:2/64 scope link
           valid_lft forever preferred_lft forever

在容器内部是 `eth0`，并且有一个 `172.16.0.2/16` 的 ip 地址，就是前面 `docker0` 管理的网段。这个网络在主机上不可见的，因为它在自己的 network namespace 里面（namespace 是 docker 实现的原理之一，可以简单理解为独立的沙盒）：

    ➜  ~ docker inspect -f '{{ .State.Pid }}' 5e03
    8820
    ➜  ~ sudo ls /proc/8820/ns/ -lh
    total 0
    lrwxrwxrwx 1 root root 0 May 30 15:03 ipc -> ipc:[4026532552]
    lrwxrwxrwx 1 root root 0 May 30 15:03 mnt -> mnt:[4026532550]
    lrwxrwxrwx 1 root root 0 May 30 15:00 net -> net:[4026532573]
    lrwxrwxrwx 1 root root 0 May 30 15:03 pid -> pid:[4026532553]
    lrwxrwxrwx 1 root root 0 May 30 16:38 user -> user:[4026531837]
    lrwxrwxrwx 1 root root 0 May 30 15:03 uts -> uts:[4026532551]

找到容器在主机上的 Pid，就能看到它所有的 namespace。`ip netns list` 看不到 docker 的网络命名空间，可以参考 [stackoverflow 这个问题](http://stackoverflow.com/questions/31265993/docker-networking-namespace-not-visible-in-ip-netns-list)。

目前还没有很好的办法查看哪个 vethXXX 对应了哪个容器，希望以后能有工具负责这一层的管理。

容器内部的路由是这样的：

    / # ip route
    default via 172.17.0.1 dev eth0
    172.17.0.0/16 dev eth0  src 172.17.0.2

默认的路由会发送到 `172.17.0.1` 也就是 `docker0` 的地址，然后 docker0 通过主机的 eth0 发送出去。前提是主机配置了自动转发网络报文：

    # cat /proc/sys/net/ipv4/ip_forward
    1

下面就分析一下具体的报文是怎么发送到外面的：

1. 容器内部发送一条报文，查看路由规则，默认转发到 `172.17.0.1`（如果是同一个网段，会直接把源地址标记为 `172.17.0.2` 进行发送）
2. 通过 `eth0` 发送的报文，会在 `vethXXX` 被接收，因为它直接连在 `docker0` 上，所以默认路由到 `docker0`
3. 这个时候报文已经来到了主机上，查询主机的路由表，发现报文应该通过 eth0 从默认网关发送出去，那么报文就被转发给 eth0（就是前面提到的要打开 linux 系统的自动转发配置）
4. 匹配机器上的 iptables，发现有一条 `-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE`，也就是 SNAT 规则，那么 linux 内核会修改 ip 源地址为 eth0 的地址，维护一条 NAT 规则记录，然后把报文转发出去。（也就是说对于外部来说，报文是从主机 eth0 发送出去的，无法感知容器的存在）


### 怎么访问容器提供的服务？

通过上面一段的解释，你会明白很重要的一个点：**外部是无法感知到容器内部的网络的，或者说容器的网络是被主机 eth0 网络封装起来的。**那么一个自然的问题就是：容器怎么提供服务供外部访问？

答案只有四个字：端口映射。

所有容器要提供服务，必须要把服务的端口映射到主机的某个端口上，所有的报文都是通过主机进行转发的。具体是怎么做的呢？我们来创建一个 nginx 容器试试：

    docker@default:~$ docker run -d -P nginx
    69635982fb337901bfa070507d5b296b7d991d9391a82f51a5bd210da248b27a

    docker@default:~$ docker ps
    CONTAINER ID        IMAGE                           COMMAND                  CREATED             STATUS              PORTS                                           NAMES
    69635982fb33        nginx                           "nginx -g 'daemon off"   2 seconds ago       Up 2 seconds        0.0.0.0:32769->80/tcp, 0.0.0.0:32768->443/tcp   insane_payne

上面的 `-P` 参数就是告诉 docker 要把容器暴露的端口映射到主机上。用 `docker ps` 也可以看到 `0.0.0.0:32769->80/tcp, 0.0.0.0:32768->443/tcp`，说明容器暴露了两个端口 80 和 443 到主机上。也可以通过 `docker port` 命令查看：

    ➜  ~ docker port 6963
    443/tcp -> 0.0.0.0:32768
    80/tcp -> 0.0.0.0:32769

其它网络方面的配置和之前相同，使用 `iptables-save` 命令发现规则列表中多了和端口有关的内容：


    -A DOCKER ! -i docker0 -p tcp -m tcp --dport 32768 -j DNAT --to-destination 172.17.0.2:443
    -A DOCKER ! -i docker0 -p tcp -m tcp --dport 32769 -j DNAT --to-destination 172.17.0.2:80

    -A DOCKER -d 172.17.0.2/32 ! -i docker0 -o docker0 -p tcp -m tcp --dport 443 -j ACCEPT
    -A DOCKER -d 172.17.0.2/32 ! -i docker0 -o docker0 -p tcp -m tcp --dport 80 -j ACCEPT

前面两条规则的意思是：从 eth0 接收到的端口为 32768/32769 的报文，修改目的 ip 地址之后转发给容器内部的服务端口；后面两个规则的意思是：接受发送给容器 80 和 443 端口的报文。

这样的话，外部可以通过访问主机的 32769 端口来获取容器的服务：

    ➜  ~ dm ip default
    192.168.99.100

    ➜  ~ http -v http://192.168.99.100:32769
    GET / HTTP/1.1
    Accept: */*
    Accept-Encoding: gzip, deflate
    Host: 192.168.99.100:32769
    User-Agent: HTTPie/0.8.0


    HTTP/1.1 200 OK
    Accept-Ranges: bytes
    Connection: keep-alive
    Content-Length: 612
    Content-Type: text/html
    Date: Tue, 31 May 2016 05:58:27 GMT
    ETag: "5744a32f-264"
    Last-Modified: Tue, 24 May 2016 18:53:35 GMT
    Server: nginx/1.11.0

    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
        body {
            width: 35em;
            margin: 0 auto;
            font-family: Tahoma, Verdana, Arial, sans-serif;
        }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>

    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>

    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>

### 主机上容器之间怎么通信的？

这个部分我们来看看同一台主机上面，多个容器之间是怎么通信的。有了上面的知识，这个就简单很多了。

比如我们运行了两个容器 busybox 和 nginx，ip 分别是 172.17.0.2 和 172.17.0.4。我们 `docker exec` 到 busybox 中，ping nginx 容器：

    # ping 172.17.0.4
    PING 172.17.0.4 (172.17.0.4): 56 data bytes
    64 bytes from 172.17.0.4: icmp_seq=0 ttl=64 time=0.094 ms
    64 bytes from 172.17.0.4: icmp_seq=1 ttl=64 time=0.075 ms

网络报文从容器发送到 `docker0` 之后，主机上有这么一条路由规则：

    172.17.0.0/16 dev docker0  proto kernel  scope link  src 172.17.0.1

`docker0` 就是一台交换机，它记录了上面连接所有的容器 ip 和 mac 地址的对应关系。
当发现报文是自己管理的网段时，`docker0` 直接把报文转发到对应 ip 连接的 vethXXX 接口，然后容器里的 eth0 就看到报文了。


## 参考资料

- [Docker Network configuration](https://docs.docker.com/v1.7/articles/networking/)
- [Docker网络详解及pipework源码解读与实践](http://www.infoq.com/cn/articles/docker-network-and-pipework-open-source-explanation-practice)
