---
layout: post
title: "在 mac 系统配置 docker 开发环境"
excerpt: "使用 docker 提供的 toolbox 搭建 mac 上的开发环境，通过 virtualbox 和 docker-machine 运行第一个容器。"
categories: blog
tags: [docker, mac, virtualbox]
comments: true
share: true
---

在这篇文章中，我们将介绍如何在 mac 平台下使用 docker toolbox 中提供的工具来运行第一个 docker 容器。

## 在 Mac 上安装 docker 环境

docker 提供了 toolbox 方便 windows 和 mac 下运行容器，主要的功能组件有：

- [docker-machine](https://github.com/docker/machine)：管理 docker 主机，支持在 virtualbox、digitalocean、azure、rackspace 等上面直接创建一个安装了 docker 的主机。**我们主要使用这个命令在 virtualbox 上管理机器。**
- [docker-compose](https://github.com/docker/compose)：管理和运行多个容器应用的工具
- docker-client：运行在 mac 上的 docker 客户端，所有的操作都是通过客户端来运行的，客户端和 docker daemon 打交道来完成实际的工作
- kitematic：图形化的容器管理界面，目前还是 beta 版本（这篇文章不会介绍）。

这些组件的关是这样的：通过 docker-machine 来生成和管理 docker host（实际运行 docker daemon和跑容器的机器），然后通过 docker-client 和不同的 host 交互。示意图如下：

![](https://docs.docker.com/engine/installation/images/mac_docker_host.svg)

toolbox 在 mac 下的安装请参考[官方文档](https://docs.docker.com/engine/installation/mac/)，因为提供的图形化安装比较简单，这里就不赘述。

**NOTE：这篇文章中，所有 docker host 都运行在 virtualbox 上，请保证你已经安装了 virtualbox 最新版！**

## 创建第一个容器

如果你运行了 kitematic，那么它已经为你创建了一个 host：

    ➜  ~ docker-machine ls
    NAME      ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER    ERRORS
    default   -        virtualbox   Running   tcp://192.168.99.100:2376           v1.11.1

如果没有的话，我们就自己创建第一个机器：

    ➜  ~ docker-machine create -d virtualbox box
    Running pre-create checks...
    Creating machine...
    (box) Copying /Users/cizixs/.docker/machine/cache/boot2docker.iso to /Users/cizixs/.docker/machine/machines/box/boot2docker.iso...
    (box) Creating VirtualBox VM...
    (box) Creating SSH key...
    (box) Starting the VM...
    (box) Check network to re-create if needed...
    (box) Waiting for an IP...
    Waiting for machine to be running, this may take a few minutes...
    Detecting operating system of created instance...
    Waiting for SSH to be available...
    Detecting the provisioner...
    Provisioning with boot2docker...
    Copying certs to the local machine directory...
    Copying certs to the remote machine...
    Setting Docker configuration on the remote daemon...
    Checking connection to Docker...
    Docker is up and running!
    To see how to connect your Docker Client to the Docker Engine running on this virtual machine, run: docker-machine env box

上面的命令就是通过 virtualbox 安装 boot2docker.iso 这个镜像，最后一个参数就是机器的名称 `box` 。输出的最后一行，有一个提示命令，我们来运行一下：

    ➜  ~ docker-machine env box
    export DOCKER_TLS_VERIFY="1"
    export DOCKER_HOST="tcp://192.168.99.101:2376"
    export DOCKER_CERT_PATH="/Users/cizixs/.docker/machine/machines/box"
    export DOCKER_MACHINE_NAME="box"
    # Run this command to configure your shell:
    # eval $(docker-machine env box)

可以看到输出的是一些 docker 有关的环境变量。要想使用这个 docker daemon 来管理容器的话，把这些变量 export 到终端：

    eval $(docker-machine env box)
    ➜  ~ env | grep -i docker
    DOCKER_TLS_VERIFY=1
    DOCKER_HOST=tcp://192.168.99.101:2376
    DOCKER_CERT_PATH=/Users/cizixs/.docker/machine/machines/box
    DOCKER_MACHINE_NAME=box

然后我们运行 docker 命令的时候，docker client 就会从环境变量中读取 docker host 的连接信息，和它进行通信：

    ➜  ~ docker images
    REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
    ➜  ~ docker ps
    CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
    ➜  ~ docker version
    Client:
     Version:      1.11.1
     API version:  1.23
     Go version:   go1.5.4
     Git commit:   5604cbe
     Built:        Tue Apr 26 23:44:17 2016
     OS/Arch:      darwin/amd64

    Server:
     Version:      1.11.1
     API version:  1.23
     Go version:   go1.5.4
     Git commit:   5604cbe
     Built:        Wed Apr 27 00:34:20 2016
     OS/Arch:      linux/amd64

这样呢，我们就能运行自己的第一个容器：

    ➜  ~ docker run -d busybox top
    Unable to find image 'busybox:latest' locally
    latest: Pulling from library/busybox

    385e281300cc: Pull complete
    a3ed95caeb02: Pull complete
    Digest: sha256:4a887a2326ec9e0fa90cce7b4764b0e627b5d6afcb81a3f73c85dc29cea00048
    Status: Downloaded newer image for busybox:latest
    656f4ec68b96a12f0a1fd92ef9262a6ef95f80cc3a5ea10749a40c1323b33926

嗯，熟悉的下载镜像和运行容器的输出！

**NOTE: 当我们有多个 docker host 的时候，每次都要通过 `eval $(docker-machine env name)` 来切换。**

## docker 网络模式

### docker-machine 网络

可以使用 `docker-machine config ` 命令查看 docker 主机的配置：

    ➜  ~ dm config box
    --tlsverify
    --tlscacert="/Users/cizixs/.docker/machine/certs/ca.pem"
    --tlscert="/Users/cizixs/.docker/machine/certs/cert.pem"
    --tlskey="/Users/cizixs/.docker/machine/certs/key.pem"
    -H=tcp://192.168.99.101:2376

我们看最后一行，前面的 ip 就是主机的地址，也就是说我们的主机是监听在它所在地址的 2376 端口的。通过 `docker-machine ssh box` 进入到主机，可以查看它的网络信息：

    docker@box:~$ ip addr
    3: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:a1:7a:a4 brd ff:ff:ff:ff:ff:ff
        inet 10.0.2.15/24 brd 10.0.2.255 scope global eth0
           valid_lft forever preferred_lft forever
        inet6 fe80::a00:27ff:fea1:7aa4/64 scope link
           valid_lft forever preferred_lft forever
    4: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:a5:3e:71 brd ff:ff:ff:ff:ff:ff
        inet 192.168.99.101/24 brd 192.168.99.255 scope global eth1
           valid_lft forever preferred_lft forever
        inet6 fe80::a00:27ff:fea5:3e71/64 scope link
           valid_lft forever preferred_lft forever

我们看到 `eth1` 的地址就是 `192.168.99.101`，而上面的配置就是 `docker daemon` 运行的参数：

    docker@box:~$ ps aux | grep docker | grep daemon
    root      2618  0.0  3.8 431372 38948 ?        Sl   02:20   0:02 /usr/local/bin/docker daemon -D -g /var/lib/docker -H unix:// -H tcp://0.0.0.0:2376 --label provider=virtualbox --tlsverify --tlscacert=/var/lib/boot2docker/ca.pem --tlscert=/var/lib/boot2docker/server.pem --tlskey=/var/lib/boot2docker/server-key.pem -s aufs

一个需要注意的的问题是：virtualbox 创建出来的虚拟机有两个网卡信息。`eth0` 是正常访问主机的网络接口，模式为 `NAT`；`eth1` 是和 docker 通信的网络接口，网络模式是 `Host-only Adapter`。

到此，我们就搭建了自己的开发环境。
