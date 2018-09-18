---
layout: post
title: "使用 coreos 运行 docker 的 netns 问题"
excerpt: "在 vagrant 开发环境运行 docker 发现无法查看 netns 的内容，遇到 netns invalid parameter 的错误，这篇文章讲述了如何解决这个问题。"
categories: blog
tags: [docker, container, coreos, netns, nsenter, iproute2]
comments: true
share: true
---

## 背景知识

`docker` 把所有的 network namespace 放在 `/var/run/docker/netns` 目录下面，要进入到 network namespace 有两种方法：

- 使用 `nsenter`：`nsenter --net=/var/run/docker/netns/<uuid> ip addr` ，可以使用 `nsenter --net=/var/run/docker/netns/<uuid> sh` 进入到该 ns 的 shell 中，后续所有的命令都是在这个 ns 中执行的
- 使用 `ip` 命令，首先要把 docker ns 链接到 `ip` 命令管理的地方 `ln -s /var/run/docker/netns/ /var/run/netns`，然后就可以使用 `ip netns exec <uuid> ip addr` 来查看某个 ns 里面的网络设置

## 问题描述

但是我在自己 coreos 的开发环境中，却出现了无法使用上述方式进入到 ns 的情况。使用两种方法出现的错误如下，


`nsenter` 命令错误：

```bash
core@core-01 ~ $ sudo nsenter --net=/var/run/docker/netns/62c9960f834f ip addr
nsenter: reassociate to namespace 'ns/net' failed: Invalid argument
```

`ip netns` 命令错误：

```bash
core@core-01 ~ $ sudo ip netns exec da1ffcf7ed0f ip addr
setting the network namespace "da1ffcf7ed0f" failed: Invalid argument
```

两种方法都是 `invalid argument`，没有进一步的报错信息。

coreos 的版本信息：

```bash
core@core-01 ~ $ cat /etc/os-release
NAME=CoreOS
ID=coreos
VERSION=1192.2.0
VERSION_ID=1192.2.0
BUILD_ID=2016-10-21-0026
PRETTY_NAME="CoreOS 1192.2.0 (MoreOS)"
ANSI_COLOR="1;32"
HOME_URL="https://coreos.com/"
BUG_REPORT_URL="https://github.com/coreos/bugs/issues"
```

docker 版本信息：

```bash
core@core-01 ~ $ docker version
Client:
 Version:      1.12.1
 API version:  1.24
 Go version:   go1.6.3
 Git commit:   7a86f89
 Built:
 OS/Arch:      linux/amd64

Server:
 Version:      1.12.1
 API version:  1.24
 Go version:   go1.6.3
 Git commit:   7a86f89
 Built:
 OS/Arch:      linux/amd64
```

## 解决方案

从网络上搜索了蛮久，找到了[这个问题的解决方案](https://forums.docker.com/t/unable-to-check-docker-overlay-network-namespace/17267)。

办法也比较简单：注释掉 `docker.service` 文件 `MountFlags=slave` 那行配置信息就 ok 了。具体的原因是这样的：这个参数是告诉 `systemd` 用什么方式 mount 要启动 process root 的 root filesystem 的，[`salve` 模式](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt) 下宿主机（host）看不到该进程的文件。所以才会报之前的错误！（ network namesapce 的报错信息真的很不清晰，完全看不出来问题）

按理说，通过 `systemctl status docker` 找到 docker 服务的 `docker.service` 文件位置，然后直接修改就行了。但是该文件所在的路径 `/usr/lib/systemd/system/docker.service` 是只读文件系统，需要把它拷贝到 `/etc/systemd/system` 目录下就行修改，然后执行：

```bash
systemctl daemon-reload
systemctl restart docker
```

修改的内容就生效了。具体的配置可以参考 coreos 的[官方文档](https://coreos.com/os/docs/latest/customizing-docker.html) **Enabling the Docker debug flag** 部分。

所有的操作执行结束后，我们就能看到期望的运行结果：

```bash
core-02 core # ip netns exec cccc9409d140 ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
20: eth0@if21: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether 02:42:0a:ff:00:04 brd ff:ff:ff:ff:ff:ff
    inet 10.255.0.4/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:aff:feff:4/64 scope link
       valid_lft forever preferred_lft forever
22: eth1@if23: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:13:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.19.0.2/16 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe13:2/64 scope link
       valid_lft forever preferred_lft forever
```
