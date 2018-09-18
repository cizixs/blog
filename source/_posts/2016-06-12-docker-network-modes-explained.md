---
layout: post
title: "docker 容器的网络模式"
excerpt: "容器启动的时候除了默认网络之外，还有 host、container、none 三种网络模式可选，这篇文件就介绍这三种模式的用法。"
categories: blog
tags: [docker, network, container, iptables]
comments: true
share: true
---

## 简介

docker 默认提供了四种模式，供容器启动的时候选择：bridge、none、container、host。[前一篇文章](http://cizixs.com/2016/06/01/docker-default-network)已经介绍了 bridge 模式，这篇文章就介绍剩下的三种模式，已经它们可能的使用场景。

![](http://developerblog.info/content/images/2015/11/docker-turtles-communication.jpg)

## 测试环境

说明一下这篇文章中所有测试的基本配置和环境信息：

所有测试都是运行在 Mac 上的 virtualbox 虚拟机中，使用 docker toolbox 管理 docker 主机，使用说明可以参考[之前的文章](http://cizixs.com/2016/05/31/install-docker-dev-environment-on-mac)。

- Mac OS
- virtualbox 5.0.20
- docker 1.11.2

先创建一台主机：

    docker-machine create -d virtualbox default

这台主机的网络信息如下：

    root@default:/home/docker# ip addr
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
           valid_lft forever preferred_lft forever
    2: dummy0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default qlen 1000
        link/ether d2:b1:81:17:b5:de brd ff:ff:ff:ff:ff:ff
    3: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:9e:d2:22 brd ff:ff:ff:ff:ff:ff
        inet 10.0.2.15/24 brd 10.0.2.255 scope global eth0
           valid_lft forever preferred_lft forever
        inet6 fe80::a00:27ff:fe9e:d222/64 scope link
           valid_lft forever preferred_lft forever
    4: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:ba:e9:64 brd ff:ff:ff:ff:ff:ff
        inet 192.168.99.134/24 brd 192.168.99.255 scope global eth1
           valid_lft forever preferred_lft forever
        inet6 fe80::a00:27ff:feba:e964/64 scope link
           valid_lft forever preferred_lft forever
    6: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
        link/ether 02:42:79:bf:98:26 brd ff:ff:ff:ff:ff:ff
        inet 172.17.0.1/16 scope global docker0
           valid_lft forever preferred_lft forever

除了有 `eth0` 和 `eth1` 之外，还有用来管理容器网络的 `docker0`。

下面所有的容器都是运行在这台主机上的！

### host 模式

在这个模式下，docker 不会为容器创建单独的网络 namespace，而是共享主机的 network namespace，也就是说：容器可以直接访问主机上所有的网络信息。

让容器运行在 host 模式很简单：启动容器的命令行添加 `--net=host` 参数就搞定了！

    # docker run -d --name=busybox --net=host busybox top
    6be9f72531705549169fd77236e36a79d2758e08ba1f8b6f431f9f0e40e7245d

我们来看一下容器的网络配置，发现和之前主机上看到的一模一样：

    root@default:/home/docker# docker exec busybox ip addr
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
           valid_lft forever preferred_lft forever
    2: dummy0: <BROADCAST,NOARP> mtu 1500 qdisc noop qlen 1000
        link/ether d2:b1:81:17:b5:de brd ff:ff:ff:ff:ff:ff
    3: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
        link/ether 08:00:27:9e:d2:22 brd ff:ff:ff:ff:ff:ff
        inet 10.0.2.15/24 brd 10.0.2.255 scope global eth0
           valid_lft forever preferred_lft forever
        inet6 fe80::a00:27ff:fe9e:d222/64 scope link
           valid_lft forever preferred_lft forever
    4: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
        link/ether 08:00:27:ba:e9:64 brd ff:ff:ff:ff:ff:ff
        inet 192.168.99.134/24 brd 192.168.99.255 scope global eth1
           valid_lft forever preferred_lft forever
        inet6 fe80::a00:27ff:feba:e964/64 scope link
           valid_lft forever preferred_lft forever
    6: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue
        link/ether 02:42:79:bf:98:26 brd ff:ff:ff:ff:ff:ff
        inet 172.17.0.1/16 scope global docker0
           valid_lft forever preferred_lft forever

那么应用场景呢？既然是共享了网络，自然也就是容器需要直接使用到主机网络的情况，比如启动一个 [tcpdump 的容器](https://hub.docker.com/r/corfr/tcpdump/)抓取主机上的网络报文:

    $ docker run --net=host -v $PWD:/data corfr/tcpdump -i any -w /data/dump.pcap "icmp"

这种模式最大的缺点是：容器都是直接共享主机网络空间的，如果出现任何的网络冲突都会出错，比如**在这个模式下无法启动两个都监听在 80 端口的容器**。

### container 模式

这个模式下，一个容器直接使用另外一个已经存在容器的网络配置：ip 信息和网络端口等所有网络相关的信息都是共享的。**需要注意的是：这两个容器的计算和存储资源还是隔离的。**

kubernetes 的 pod 就是用这个实现的，同一个 pod 中的容器共享一个 network namespace。

这个试验中，我们会运行两个 nginx 容器：web1 和 web2：

- web1 监听在 80 端口，使用默认的网络模型
- web2 坚挺在 8080 端口，使用 container 网络模型共享 web1 的网络

先启动 web1，通过 port mapping 把端口绑定到主机上：

    root@default:/home/docker# docker run -d --name=web1 -p 80:80 nginx
    d1bb662da72a0141906de6bfd920221d4d4f1f6dd4ccf764294e4a7dc4cb7333

使用 curl 命令验证容器运行正常：

    root@default:/home/docker# curl http://localhost:80
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx on Debian!</title>
    <style>
        body {
            width: 35em;
            margin: 0 auto;
            font-family: Tahoma, Verdana, Arial, sans-serif;
        }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx on Debian!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working on Debian. Further configuration is required.</p>

    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a></p>

    <p>
          Please use the <tt>reportbug</tt> tool to report bugs in the
          nginx package with Debian. However, check <a
          href="http://bugs.debian.org/cgi-bin/pkgreport.cgi?ordering=normal;archive=0;src=nginx;repeatmerged=0">existing
          bug reports</a> before reporting a new bug.
    </p>

    <p><em>Thank you for using debian and nginx.</em></p>


    </body>
    </html>

看一下里面的网络配置：

    root@default:/home/docker# docker exec web1 ip addr
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
           valid_lft forever preferred_lft forever
    21: eth0@if22: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
        link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
        inet 172.17.0.2/16 scope global eth0
           valid_lft forever preferred_lft forever
        inet6 fe80::42:acff:fe11:2/64 scope link
           valid_lft forever preferred_lft forever

第二个容器也是 nginx 容器，不同的是：它监听在 8080 端口。我们直接使用 `-v` 参数来覆盖掉 nginx 的配置文件，达到监听不同端口的效果。和 host 模式相同，使用 `--net` 参数让新建的容器使用 `web1` 的网络：

    docker run --name=web2 -v ${PWD}/default.conf:/etc/nginx/sites-available/default -v ${PWD}/index.html:/var/www/html/index.html -d --net=container:web1 -p 8080:8080 nginx

`default.conf` 文件就是修改了 nginx 默认配置文件的端口，把它变成 8080；`inedx.html` 可以随便修改一点，以区别于默认的内容。

如果执行上面命令的话，docker 会报错：

    docker: Error response from daemon: Conflicting options: port publishing and the container type network mode.
    See 'docker run --help'.

提示端口转发不能和 container 模式一起使用，删除 `-p 8080:8080` 重新运行，一起正常。但是无法从主机上直接访问 web2 的服务，因为服务端口没有被转发出来：

    root@default:/home/docker# curl http://127.0.0.1:8080
    curl: (7) Failed connect to 127.0.0.1:8080; Connection refused

进入到 web2 中，可以看到网络配置和 web1 一致：

    root@default:/home/docker# docker exec -it web2 bash
    [ root@d1bb662da72a:/etc/nginx ]$ ip addr
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
           valid_lft forever preferred_lft forever
    21: eth0@if22: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
        link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
        inet 172.17.0.2/16 scope global eth0
           valid_lft forever preferred_lft forever
        inet6 fe80::42:acff:fe11:2/64 scope link
           valid_lft forever preferred_lft forever

在容器里面可以验证 nginx 服务是正常的：

    [ root@d1bb662da72a:/etc/nginx ]$ curl http://127.0.0.1:8080
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx on Debian!</title>
    <style>
        body {
            width: 35em;
            margin: 0 auto;
            font-family: Tahoma, Verdana, Arial, sans-serif;
        }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx 8080 on Debian!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working on Debian. Further configuration is required.</p>

    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a></p>

    <p>
          Please use the <tt>reportbug</tt> tool to report bugs in the
          nginx package with Debian. However, check <a
          href="http://bugs.debian.org/cgi-bin/pkgreport.cgi?ordering=normal;archive=0;src=nginx;repeatmerged=0">existing
          bug reports</a> before reporting a new bug.
    </p>

    <p><em>Thank you for using debian and nginx.</em></p>


    </body>
    </html>

为什么 `web2` 容器不允许端口转发呢？当然出于安全和权限控制的角度，不然的话，一个正在运行的容器网络配置可以被任意一个容器修改，这是很危险的行为。那怎么把 `web2` 的服务转发出来呢？答案是通过 `web1`：因为它们两个的网络是共享的。我们需要在运行 `web1` 的时候就指定好所有要映射出来的端口。

把之前的容器删除，按照下面的命令重新运行就搞定了：

    root@default:/home/docker# docker run -d --name=web1 -p 80:80 -p 8080:8080 nginx
    root@default:/home/docker# docker run --name=web2 -v ${PWD}/default.conf:/etc/nginx/sites-available/default -v ${PWD}/index.html:/var/www/html/index.html -d --net=container:web1 nginx

那么这种网络模式的应用场景是什么呢？我们还是从特性出发：两个或多个容器共享一个网络空间，包括 ip 和端口等信息。可以类比这种模型到传统虚拟机上面运行的多个应用：多个服务，但是它们共享机器的网络。比如：有一个应用提供 API，另一个应用是它的命令行工具，那么后者可以共享前者的网络。用户只需要访问命令行工具，不用直接和 HTTP 打交道。

### none 模式

这种模式正如它的名字说明的那样：不做配置。**容器有自己的 network namespace，但是没有做任何的网络配置**。

选择这种模式，一般是用户对网络有自己特殊的需求，不希望 docker 预设置太多的东西。


## 其他

### link 模式

docker 还提供了 link 模式，让容器之间能够通过名称访问。通过设置环境变量和修改 `/etc/hosts` 文件实现的，算是本地的服务发现机制吧。

不过自从 docker 1.9 网络模型出来之后（可以看看到这个 [issue](https://github.com/docker/docker/issues/9983) 了解网络模型提出来的背景），这个特性会用到的越来越少。


## 参考资料

- [DockOne技术分享（五）：Docker网络详解及Libnetwork前瞻](http://dockone.io/article/402)
- [docker 网络模型初次被提出来的 issue](https://github.com/docker/docker/issues/8951)
