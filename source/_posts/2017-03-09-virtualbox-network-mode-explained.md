---
layout: post
title: "virtualbox 常用网络模式解释和配置"
excerpt: "Virtualbox 可以方便地让我们在一台机器上创建多个虚拟机，默认网络下，虚拟机就可以访问外网。如果对网络有特殊的需求，就需要了解各种网络模式的功能和区别。"
categories: blog
tags: [virtualbox network]
comments: true
share: true
---

## 介绍

默认情况下，virtualbox 创建的虚拟机网络为 `NAT` 模式，这样虚拟机就可以通过主机的网络连接到外部网络。如果需要直接访问虚拟机网络，可以通过端口转发机制。

但是除了默认的网络配置，virtualbox 还提供了其他网络选项来实现用户各种各样的需求。创建网络的时候，有两类配置要做：

- 网卡的硬件类型
- 网卡的网络模式

virtualbox 每台虚拟机支持多块虚拟网卡，每块网络都可以配置不同的网络模式。

## 网卡类型

virtualbox 支持多种虚拟网卡类型，选择对应的网卡需要主机有对应的网卡驱动。如果对网卡没有特殊的需求，使用默认就可以了。

- AMD PCNet PCI II (Am79C970A)
- AMD PCNet FAST III (Am79C973, the default)
- Intel PRO/1000 MT Desktop (82540EM)
- Intel PRO/1000 T Server (82543GC)
- Intel PRO/1000 MT Server (82545EM)
- Paravirtualized network adapter (virtio-net)

## 四种模式

每块网卡都可以运行在不同的网络模式下，它们的功能不太相同，这个部分就介绍不同网络模式的区别。我们会介绍命令行的操作实现，每种模式都可以通过 virtualbox GUI 界面配置，因为配置起来比较简单，这篇文章将不会介绍具体步骤。

### Not Attached

有网卡，但是没有连接网线。这种情况类似于服务器没有连接网线的情况，因此也就无法访问网络。一般不会使用这种情况，也没有需要特殊说明的地方。

### Network Address Translatoin(NAT)

![](http://78rcxs.com1.z0.glb.clouddn.com/NAT.png)

默认的网络模式，不需要任何主机或者虚拟机上的设置。能满足虚拟机连接外网的需求，可以简单认为 virtualbox 充当了虚拟机和外部网络的路由器，会自动转发虚拟的报文。

虚拟机发出去的报文，会被 virtualbox NAT 引擎拦截，抽取其中的 TCP/IP 数据，然后用主机的网络进行发送，对于外部网络来说，它们看到这个报文是主机发送的。virtualbox 还会自动监听响应报文，然后把目标地址修改成虚拟机的地址，这样虚拟机就能收到应答报文。

这个模式下，网络的 ip 是通过 virtualbox 自带的 DHCP Server 分配的。virtualbox 会保证虚拟机 ip 和 主机不冲突，默认情况下，第一个网卡的地址范围是 `10.0.2.0/24`，第二个网卡的地址范围是 `10.0.3.0/24`，以此类推。如果需要修改这个地址范围，可以通过 `VBoxManage` 命令

```bash
VBoxManage modifyvm "vmname" --natnet1 "192.168/16"`
```

这条命令行就是把 vmname 虚拟机的第一个 nat 网卡地址修改为 `192.168/16` 网段。

举个例子：在我本地机器上，NAT 网卡的地址是 `10.0.2.15/24`，网关地址是 `10.0.2.2`，DHCP server 的地址是 `10.0.2.3`。

虚拟机之间的网络是不连通的，而且外部网络无法直接访问虚拟机网络。如果需要在虚拟机里提供服务，需要使用端口转发的功能。端口转发：如果要访问虚拟机的某个端口， virtualbox 会在主机上选择一个端口和虚拟机端口进行匹配（两个端口可以不同），要访问虚拟机服务，直接访问主机 ip 地址和主机开放的端口就行。比如一般情况下，虚拟机会通过开启 ssh 的端口转发，方便从主机上通过 ssh 连接到虚拟机。

端口转发需要使用者记住端口的使用情况，只要端口没有被占用就能进行转发。可以通过命令行添加端口转发的规则：

```bash
VBoxManage modifyvm "VM name" --natpf1 "guestssh,tcp,,2222,,22"
```

比如这条命令就是给虚拟机添加一条规则，所有发到主机 `2222` 端口的报文都要转发到虚拟机的 `22` 端口。`--natpf` 后面的数字代表这个规则是作用于第一块网卡的，最后一个参数的各个字段含义是：

1. 转发规则的名字。如果没有指定，virtualbox 会自动生成一个名字
2. 报文类型。支持 tcp、udp
3. 主机地址。如果不填写，不论主机哪个网卡收到 `2222` 都要转发；如果有填写，则指定某个网卡收到 `2222` 端口的报文才转发。比如 `127.0.0.1,2222` 说明只转发本地的报文
4. 主机端口。监听的主机端口
5. 虚拟机地址。如果虚拟机不是 DHCP server 分配的，需要告诉 virtualbox 虚拟机网卡的 ip 地址
6. 虚拟机端口。虚拟机内部提供服务的端口，报文会转发到这个端口

删除端口转发规则也是类似，区别是提供规则名称就行：

```bash
VBoxManage modifyvm "VM name" --natpf1 delete "guestssh"
```

**NOTE**：不要指定主机的小于 1024 的端口号，因为在 linux 机器上只有 root 权限才能使用这些端口。容易造成冲突，而且权限问题会导致虚拟机无法启动。

### Bridge Networking

![](http://78rcxs.com1.z0.glb.clouddn.com/Bridged.png)

bridged 网络模式是把虚拟机放到和主机同一个网络上。每个虚拟机必须要指定某个网卡作为父亲节点。

虚拟机的网络和主机的网络完全一样，如果主机是 DHCP 分配的 ip，虚拟机也需要自动去 DHCP server 分配 ip 地址。

虚拟机之间能够互相联通，虚拟机和主机也是直接联通的，不需要任何的转发。

### Internal Networking

![](http://78rcxs.com1.z0.glb.clouddn.com/Internal.png)

Internal 网络模式和 Bridged 模式很相似，只不过这些虚拟机只能和同主机通模式下的其他虚拟机联通，无法访问外网。

这种模式有个很好的优势就是安全性比较高，主机是无法直接看到虚拟机网络报文的。

virtualbox 就是个交换机，连接着主机上该网络模式下连接到同个 internal 网络的虚拟机。虚拟机之间通过 virtualbox 之间通信，virtualbox 不会把报文发送到外部。

网络 Internal 是通过名字识别的，添加网卡的时候必须选择网络名字，网络名称相同的虚拟机会自动连接到同一个 Internal 网络。

如果希望通过命令行来实现也是可以的：

```bash
VBoxManage modifyvm "VM name" --nic<x> intnet
```

这个命令会把虚拟机的第 `x` 块网卡加入到默认的 internal 网络 `intnet` 中。

或者你也可以指定一个 internal network 名字

```bash
VBoxManage modifyvm "VM name" --intnet<x> "networkname"
```

这个命令把虚拟机的第 `x` 块网卡设置为 internal 网络模式，并加入到 `networkname` 的网络中。

虚拟机的 ip 地址可以手动配置，也可以通过 DHCP Server 自动获取。

### Host-only Networking

![](http://78rcxs.com1.z0.glb.clouddn.com/Host-Only.png)

这种模式会在主机上创建一个虚拟的网卡（比如 `vboxnet0`），然后通过这个网卡把虚拟机连接起来。这种模式比较常用的需求是多个虚拟机需要共同协作提供服务，因为连接到相同 Host-Only 模式的虚拟机是可以直接通信的，而且主机可以直接访问这个服务，方便开发和测试。

要把虚拟机加入到 Host-Only 网络中，首先要创建对应的 Host-only 虚拟网卡。下面的命令行可以实现：

```bash
VBoxManage hostonlyif create
VBoxManage modifyvm "VM name" --nic<x> hostonly
```

第一个命令创建一个 Host-Only 网络的虚拟网卡，第二个命令把虚拟机加入到 Host-Only 网络中。

你可以手动配置 Host-Only 虚拟网卡以及虚拟机的 ip 地址，也可以使用 Virtualbox 提供的 DHCP Server：

```bash
VBoxManage dhcpserver add --ifname <hostonly_if_name> \
    --ip <dhcp-server-ip> \
    --netmask <netmask-address> \
    --lowerip <dhcp-lower-ip> \
    --upperip <dhcp-upper-ip> \
    --enable
```

## 总结

我们上面介绍了每种模式下网络的用法，看起来每种模式都很简单。强大之处在于，这些模式是可以结合起来使用的。每台虚拟机都可以配置多块网卡，根据实际的需要，把每块网络配置成不同的模式，可以形成非常复杂的网络拓扑结构。

最后，我们用一张图表总结不同模式的区别，这张表展示了不同网络模式下虚拟机各种情况的连通性：

模式 |   NAT |   Bridged |   Internal    |   Host-Only   
--- |   --- |   --- |   --- |   ---
虚拟机 -> 主机 |   可以 |   可以    |   不可以  |   可以
主机 -> 虚拟机  |   不可以  |   可以    |   不可以  |   可以
虚拟机 -> 其他主机  |   可以    |   可以    |   不可以  |   可以
其他主机 -> 主机    |   不可以  |   可以    |   不可以  |   不可以
虚拟机之间  |   不可以  |   可以    |   同网络可以  |   可以

## 参考资料

- [Virtualbox User Manual: Chapter 6. Virtual networking](https://www.virtualbox.org/manual/ch06.html)
- [Network Configuration in VirtualBox](https://www.thomas-krenn.com/en/wiki/Network_Configuration_in_VirtualBox)
- [快速理解VirtualBox的四种网络连接方式](https://www.cnblogs.com/york-hust/archive/2012/03/29/2422911.html)
