---
layout: post
title: "pxe 自动安装系统流程分析"
excerpt: "批量服务器安装系统的时候要做到自动化，这样才能提高效率和实现标准化。pxe 是 intel 开发的技术，用来引导各种操作系统（linux、windows）的安装。"
categories: blog
tags: [pxe, linux, boot, dhcp]
comments: true
share: true
---

TL;DR

本文介绍 pxe 安装 linux 系统过程中，每个阶段用到网络协议的作用和它们的数据格式。不会介绍 pxe 安装系统的原理和配置，而是假设读者已经了解这方面的信息（如果不了解的话，可以查看这篇文章： [CentOS 6.4 PXE+Kickstart unattended installation operating system](http://www.programering.com/a/MDMzcTNwATc.html)，或者网络上其他的教程）。

## 简介

![此处输入图片的描述][1]

1. 机器从网络启动，触发 pxe 相关的固件模块发送 DHCP 请求，请求以广播方式向整个网段传播
2. DHCP server 接收到请求，并响应请求，回应 ip 信息和 next_ip(tftp ip 的地址)
3. 机器配置接收到的 ip 和其他网络信息，发送 ARP 广播，获取 tftp server 的 mac 地址
4. 拿到 tftp server 的 ip 和 mac，机器就向 tftp server 请求启动脚本 `pxelinux.0`
4. tftp server 应答启动脚本 `pxelinux.0`
5. 执行 `pxelinux.0` 文件，机器把 pxe 对应的功能在内存里开始运行
5. pxe 模块根据约定，向 TFTP server 发起请求，获取启动脚本 `pxelinux.cfg`
6. 根据启动脚本的内容，请求并加载内核文件 `vmlinuz`、`initrd.img`
6. 启动安装系统流程，到指定的地址下载 kickstart（ftp/http）。
7. 根据 kickstart 文件从对应的地址（nfs/http/ftp）下载安装包安装系统，并自动配置所有的选项
8. 系统安装完成

## DHCP
机器刚启动的时候是没有网络（ip/netmask/gateway）信息的，也就是说没有办法进行网络通信。所以这时候最关键的就是获取到 ip，DHCP 协议就是完成这个功能的。DHCP 协议是 C/S 模式，服务器端配置好一段网络地址，等待客户端发送请求过来，就分配一个 ip 给它使用。

DHCP 是无连接的，底层使用 UDP 协议，使用了 67（客户端） 和 68（服务器端） 端口。
如果客户端和服务器端不在同一个网络里，将会使用到 DHCP relay 技术。

启动过程用到 DHCP 的四个步骤：

![](https://upload.wikimedia.org/wikipedia/commons/thumb/e/e4/DHCP_session.svg/260px-DHCP_session.svg.png)

1. Discovery （发现）
    因为机器刚启动，没有 ip，也不知道 dhcp server 的地址，此时的数据报是以广播的方式向机器所在的网络发送的。源地址是 `0.0.0.0`，目的地址是 `255.255.255.255`。网络中所有的机器都会收到这个请求，但是只有 dhcp server 才会作出相应。
2. Offer （提供）
    dhcp server 接收到客户端的请求，然后根据配置从维护的网络段把 IP 地址、子网掩码、网关地址等信息发送给申请的客户端。这个数据报是通过广播的形式，客户端会根据 mac 地址判断是否数据是发送给自己的。
3. Request （请求）
    接收到 offer 的客户端发送确认使用该配置的数据报给服务器端，因为某个网段可能会有多个 dhcp server。如果每个 dhcp server 同时向客户端发送应答，没有收到回复就认为客户端使用了该地址，就会造成 ip 的浪费甚至网络的错误。为了让所有的 dhcp server 都收到该请求，这个数据报也是广播。需要注意的是，此时数据报的源地址依然是 `0.0.0.0`，因为客户端还没有最终配置 ip，可以看下一步的解释。
4. Acknowledge （确认）
    服务器端接收到客户端的请求数据，向客户端发送应答数据报，授权客户端使用前一步配置的网络信息，这个数据报也是单播。然后客户端才把网络根据参数配置好！

![此处输入图片的描述][2]

除了上面用到的四个基本数据报文之外，dhcp 还有下面几种数据报类型，来实现错误处理和续租等功能：

1. NAK
    这种类型的数据报文和 ACK 相反，是上面第四步的时候，服务器端不能满足客户端使用之前网络配置的数据报。

2. Decline（拒绝）
    拒绝报文对应上面的第三步，客户端发现服务器端在第二步提供的网络信息不正确（比如 ip 已经被使用），向服务器发送的报文。

3. Information（信息）
    对应于上面第三步，客户端向服务器端请求更多的信息。

4. Release（释放）
    当客户端要终止服务器端提供的网络配置的使用，会发送该报文，以便服务器端对 ip 进行回收。

详细的 dhcp 状态转移图，来自[这篇文章](http://www.tcpipguide.com/free/t_DHCPGeneralOperationandClientFiniteStateMachine.htm)
![dhcp 状态转移图](http://www.tcpipguide.com/free/diagrams/dhcpfsm.png)

下面我们就根据实际例子中的报文分析一下 DHCP 整个过程。

首先来看一下 DHCP 数据报文的格式，关于格式的解释可以参考 [DHCP Message Format](http://www.tcpipguide.com/free/t_DHCPMessageFormat.htm) 和 wikipedia。

![](http://www.tcpipguide.com/free/diagrams/dhcpformat.png)

需要简单解释一下几个重要的选项：

+ OP: 操作码，请求是 1，应答是 2
+ HType：物理网络类型
+ HAL：物理地址长度，一般是 6
+ Hops：dhcp relay 会用到，hop count。
+ XID：交易标号，用于定位通信
+ CIADDR：客户端 IP，一般是 `0.0.0.0`，如果客户端之前有 ip，就填到这个位置
+ YIADDR：服务器端应答给客户端使用的 ip
+ SIADDR：tftp server 的 ip
+ GIADDR：DHCP relay 的 ip 地址
+ CHADDR：客户端的物理地址
+ SNAME：服务器端域名
+ Boot File：tftp 启动脚本
+ Options：选项，其他预定义的数据和

其他需要注意的事项：

1. 客户端是租期（lease period）过半的时候需要续租（release）
2. 客户端如果不续租，服务器端将回收 ip ，用于其他机器的配置
3. 客户端发送 discovery 请求没有收到回复时，将重复四次然后报错或者不断重复该步骤
4. 客户端发送 ARP 协议包来判断服务器分配给自己的 ip 有没有在使用

## ARP 协议

### ARP 介绍
机器有了自己的 ip，还知道 tftp server 的 ip，下一步就是向 tftp server 发出请求，获取启动文件。
然而仅仅知道目的机器的 ip 是不够的，网络的二层要想发送数据还需要目的机器的 mac 地址。那么知道目的机器 ip 地址后，机器还需要通过某种方式得到目的机器的 mac 地址，而根据 ip 地址获取 mac 地址就是 ARP 协议要实现的功能。

ARP 协议相对很简单，通俗地说是这样的：机器向所在的网络发送 ARP 请求广播，内容是“**谁知道 `10.3.4.5` 这个 ip 对应的 mac 地址，把它发给我**”；该网段所有机器都接收到这个请求，但是只有 `10.3.4.5` 这台机器会处理，其他机器都直接丢弃掉报文（事实上，其他机器也会根据这个报文更新自己的 ARP 缓存）。 `10.3.4.5` 这台机器接收到数据之后，发送一条单播报文，告诉请求机器 “**`10.3.4.5` 的 mac 地址是 11.22.33.44.55.66**”。

### 报文格式解析
![此处输入图片的描述][3]

### 缓存以及性能

机器每次要发送请求的时候都去获取一遍 mac 地址是很傻的做法，很明显它只要请求一次，并保存下来，下次要使用的时候就直接查询一下就可以啦。不过只要隔一段时间就更新一下，否则如果有机器的 ip 和 mac 对应关系发生变化，就会出错。

源机器保存它获取到的目的机器的 ARP 信息，反过来想，目的机器在接收到 ARP 请求的时候，也能从数据报里看到源机器的 ARP 信息。这个时候它也可以保存下来，以供后面使用，当然也要考虑到 ARP 缓存失效的问题。

其实上面也提到，不仅是目的机器收到了 ARP 请求，网段里所有的机器都能收到，这个时候所有机器都更新自己的 ARP 缓存，也是提高效率的好办法。

加上缓存功能，那么更加详细的 ARP 协议是这样的：

1. 源机器查看自己的 ARP 缓存，是否有目的机器的 mac 地址。如果有，而且没有过期，就直接使用；否则到下一步
2. 源机器构建 ARP 请求报文，并广播出去
3. 网络中所有机器都接收到该报文，并根据报文里源机器的 ip 和 mac 地址，更新自己的 ARP 缓存
4. 目的机器接收到该报文，并构建 ARP 应答报文，发送出去，目的 ip 和 mac 地址在接收到的报文里都能获取到
5. 源机器接收到 ARP 应答报文，拿到想要的 mac 地址，并根据报文里目的机器的 ip 和 mac 更新自己的 ARP 缓存

缓存机制带来的好处是节省网络流量，缩短获取目的机器 mac 地址的时间，从而加快请求的速度。当然，它也是有缺点的，那就是缓存中数据的过期问题。
当机器更换网卡、重新配置 ip、或者迁移的时候，各个机器保存的关于这台机器的 ARP 信息都是错误的。如果不能及时更新，会发生数据丢包的问题。

## TFTP（Trivial File Transfer Protocol） 协议
从名字中也可以看出，TFTP 协议比较简单。TFTP 需要服务器端和客户端两个节点，简单的流程如下：

1. 客户端发起请求，要求读服务器端的文件。此时客户端端口是随意的，记做 P1，服务器端端口是 69。
2. 服务器端驻守在 69 端口的进程收到请求，应答一个报文，表示将开始传输数据，也可能在该报文里协定数据块（data block）大小，并重新建一个端口（记做 P2）传输剩余的报文。文件以默认 512 字节 传输（此外可以通过[协商](https://tools.ietf.org/html/rfc2348)来定义每次传输的字节块）
3. 数据传输和确认通过 P1 和 P2 端口进行。每次客户端收到数据后，需要发送确认帧，服务器只有收到确认帧之后才会发送下一段数据
4. 直到服务器端接收到的数据小于规定的数据块（data block），默认是 512 字节

TFTP 协议还定义了三种传输的模式: netascii，octet，mail。

1. netascii：就是 ascii 模式，
2. octet: raw data 传输，按照字节原本的
3. mail

TFTP 底层使用的是 UDP 协议，没有差错处理和流量控制等特性。

![](http://telescript.denayer.wenk.be/~hcr/cn/idoceo/images/tftpformat.gif)


## 参考文档
+ [DHCP协议原理及其实现流程](http://blog.csdn.net/wuruixn/article/details/8282554)
+ [wikipedia Dynamic Host Configuration Protocol 页面](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol)
+ [一个网上问题的处理，引发我们对DHCP协议新的思考](http://www.h3c.com.cn/minisite/technology_circle/sutra_case/201012/703849_97665_0.htm)
+ [wireshark 官网 DHCP 介绍页面](https://wiki.wireshark.org/DHCP)


  [1]: http://images.cnitblog.com/blog/370046/201406/152331542808644.jpg
  [2]: http://i3.tietuku.com/6cce4a0f61fb6c81.png
  [3]: https://reaper81.files.wordpress.com/2010/07/arp-header3.png
