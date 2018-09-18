---
layout: post
title: "ARP 协议解析"
excerpt: "ARP 的全称是 Address Resolution Protocol，直译过来是地址解析协议。它目前最常用的场景是根据 IP 地址查询对应的 MAC 地址。"
categories: blog
tags: [linux, arp, network, wireshark, TCP/IP]
comments: true
share: true
---

## ARP 协议简介

ARP 的全称是  Address Resolution Protocol，直译过来是 **地址解析协议**。对应的 RFC 文档是 [RFC826](https://tools.ietf.org/html/rfc826)。它的作用是把 IP 地址转换为 MAC 地址。为什么需要做这件事呢？

这是因为 TCP/IP 网络协议栈是分层的，每层负责不同的功能。IP 层（layer 3）负责路由寻路，换句话说，如果目的机器和客户端不在同一个网络，IP 层会穿过错综复杂的中间网络（互联网）找到目的机器所在的网络。

当报文在某一个网络中传播时（可能源机器和目的机器本来就在同一个网络，也可能报文在路由过程中执行下一跳步骤），IP 层的功能就没有用了，这时候起作用的是 2 层网络（链路层），大多数情况下就是以太网。以太网负责把多个机器连到一起，组成一个最小单位的局域网。在以太网中，不同机器的标识是 MAC 地址，MAC 地址是机器在生产的时候厂商为机器设定的。可以使用 `ip link` 命令查看网卡的 MAC 地址，比如我的机器上这个命令的输出是：

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s25: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 50:7b:9d:ca:08:f0 brd ff:ff:ff:ff:ff:ff
```

我机器的网卡对应的 MAC 地址就是 `50:7b:9d:ca:08:f0`，这是一个 6 字节的数字，表示的时候每个字节用 `:` 分隔开，长度是 48 比特。

有了 MAC 地址，同一个以太网络上的两台机器才能够通信。机器 A 需要知道机器 B 的 MAC 地址，才能发送以太网帧；交换机收到报文之后，根据目的 MAC 地址决定应该从哪个端口发送出去；目的机器读取报文的 MAC 地址才能知道报文是不是要发给自己的。

最开始的时候，机器 A 只知道目的地址的 IP（用户用某种方式输入 IP 地址，或者通过 DNS 解析出来 IP 地址），不知道对方的 MAC 地址。这时候，机器 A 会发送 ARP 报文，去查询机器 B 的 MAC 地址，拿到 MAC 地址，就能完成通信的过程。

![](https://ws3.sinaimg.cn/large/006tNc79gy1fi242uryyqj30g90e374i.jpg)

ARP 协议的内容，以及怎么拿到 MAC 地址就是这篇文章接下来要讲解的。

## Linux ARP 缓存

如果每次主机通信都要发送 ARP 协议去查询 MAC 地址无疑是低效的，为了提供性能最常见的做法是在系统层面保存 ARP 的结果。因为 IP 地址和 MAC 地址并不会经常变化，而且主机间第一次通信之后有很大的可能性会在短时间内再次通信， 所以 ARP 缓存能够大大提高网络效率。

在 Linux 系统中，可以通过 `ip neighbour`（可以简写为 `ip neigh`） 命令管理 ARP 缓存。为了说明 Linux 系统中 ARP 的工作原理，我会在自己的机器上进行试验。试验环境如下图所示：

![](https://ws2.sinaimg.cn/large/006tNc79gy1fi28tjdwdmj30qh0fjmyk.jpg)

我的工作机器是 A，IP 地址是 `172.16.13.16`，我会 `ping` 另外一台和 A 在同一个以太网的机器 B（IP 地址是 `172.16.13.18`），并查看 A 上 ARP 缓存的情况。

两台机器在同一个子网中（连到同一台交换机上），这个可以用路由表确认：
```
➜  ~ ip route
172.16.13.0/24 dev enp0s25  proto kernel  scope link  src 172.16.13.16  metric 100 
```

在执行 `ping` 命令之前，机器 A 上并没有对方的 ARP 缓存，使用 `ip neigh` 可以列出（实际上我的机器上是有 ARP 规则的，但是这些规则都和 `172.16.13.18` 无关，因此没有打印在输出中）：

```
➜  ~ ip neigh
```

执行 `ping` 命令，呼叫对方主机。`-c 3` 表示发送多少次请求，可以看到发送 3 次请求并接收到应答后 `ping` 程序就直接退出了（如果不添加 `-c 3`，ping 会一直发送请求，除非用户使用 `ctl + C` 强制退出应用）。

```
➜  ~ ping -c 3 172.16.13.18
PING 172.16.13.18 (172.16.13.18) 56(84) bytes of data.
64 bytes from 172.16.13.18: icmp_seq=1 ttl=64 time=0.282 ms
64 bytes from 172.16.13.18: icmp_seq=2 ttl=64 time=0.238 ms
64 bytes from 172.16.13.18: icmp_seq=3 ttl=64 time=0.235 ms

--- 172.16.13.18 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2028ms
```

此时再次查看 ARP 缓存，可以看到 `172.16.13.18` 对应的表项：

```
➜  ~ sudo ip neigh
172.16.13.18 dev enp0s25 lladdr 50:7b:9d:e0:8b:a8 REACHABLE
```

这个表项表明 `172.16.13.18` 对应的 MAC 地址是 `50:7b:9d:e0:8b:a8`，`dev enp0s25` 是说发送给 `172.16.13.18` 的报文需要通过这个 interface，`REACHABLE` 表明目标地址是可达的，状态正常，其他状态还包括：

- `permanent`：这个 ARP 缓存项没有过期时间，会一直生效，直到管理员把它删除
- `noarp`：这个 ARP 表项是有效的，不需要验证它的真实性，但是它有过期时间，过期之后会被自动删除
- `stale`：ARP 表项虽然有效，但是可疑，需要发送验证 ARP 报文
- `failed`：该表项不可用，ARP 请求报文没有应答

ARP 缓存状态的详细解释可以参考[这个文档](https://people.cs.clemson.edu/~westall/853/notes/arpstate.pdf)。

## ARP 协议解析

ARP 报文格式如下图所示，对于 IP 地址转换为 MAC 地址来说，ARP 报文长度为 28 字节。

![](http://www.tcpipguide.com/free/diagrams/arpformat.png)

各个字段的含义为：

- `Hardware Type`：传输 ARP 报文的物理网络类型，最常见的是以太网类型，对应的值是 1。这个字段的长度是 2 字节
- `Protocol Type`：网络报文类型，字段长度是 2 字节。最常用的是 IPv4 报文，对应的值是 2048（十六进制为 0x0800，这和以太网帧中帧类型字段使用的 IP 报文类型相同，这是有意设计的）
- `Hardware Address Length`：物理地址长度，字段长度是 1 比特。ARP 协议报文中物理地址（MAC 地址）有多少比特，对于大部分网络来说，这个值是 6（因为 MAC 地址是 6 个字节，48 比特长）
- `Protocol Address Length`：网络地址长度，字段长度是 1 比特。表示 ARP 协议报文中网络地址（IP 地址）有多少比特，对于大部分网络来说，这个值是 4（因为 IPv4 地址是 4 个字节，32 比特）
- `Opcode`：ARP 报文的类型，接下来我们会看到几种不同的 ARP 报文。这个字段占用 2 个比特，值为 1 代表 ARP 请求报文，值为 2 代表 ARP 应答报文，3 代表 RARP 请求报文，4 代表 RARP 应答报文
- `Sender Hardware Address`：当前 ARP 报文的发送方的物理地址（MAC 地址）
- `Send Protocol Address`：当前 ARP 报文发送发的网络地址（IPv4 地址）
- `Target Hardware Address`：当前 ARP 报文接收方的物理地址（MAC 地址），如果是 ARP 情况请，这个字段为空（因为发送方就是不知道对方的 MAC 地址从会使用 ARP 来解析）
- `Target Protocol Address`：当前 ARP 报文接收方的网络地址（IPv4 地址）

需要注意的是，虽然 ARP 协议目前最常用的场景是把 IP 地址转换为 MAC 地址，但是它设计之初却是为了更一般的场景。它的硬件类型、协议类型就是为了指明要转换地址的双方；而硬件地址长度和协议地址长度指定双方的地址长度（每种协议的地址长度可以会发生变化），其对应的就是头部最后面四个地址长度。

也就是说，ARP 本身可以转换其他硬件地址和网络地址，而且允许它们的地址长度是可变的。这导致 ARP 协议现在看来是有点冗余的，毕竟 IPv4 和 MAC 地址长度都是固定的，没有必要在协议中指定。

ARP 发送出去会被封装在以太网帧里，ARP 报文中有发送端的 MAC 地址，而以太网帧的报文头部也包含了发送端的 MAC 地址，也就是说报文中有完全重复的信息。

## ARP 工作过程

**第一步**：机器 A 想和同一个以太网络的机器 B 通信，A 会现在自己的 ARP 表中查找 B 的 MAC 地址，如果能找到就直接发送以太网帧；如果没有找到，就跳到第二步
**第二步**：机器 A 发送 ARP 请求报文去查询机器 B 的 MAC 地址，这是个以太网广播报文，因此交换机会广播到网络中所有的机器
**第三步**：各个主机接收到 ARP 请求报文，如果发现 ARP 报文中询问的 IP 地址和自己不同，则直接丢弃；机器 B 发现 ARP 报文中询问的 IP 地址是自己的主机地址，则生成一个 ARP 应答报文，把自己的 MAC 地址填到报文中，发送给机器 A，这个报文是单播报文，不会发送给其他主机；同时机器 B 也会把机器 A 的 ARP 记录缓存起来
**第四步**：机器 A 接收到 B 发来的 ARP 应答，读取报文中 B 的 MAC 地址，使用这个 MAC 地址和机器 B 进行后续的通信，同时把它缓存到系统中

从机器 A `ping` 机器 B 时，用 `wireshark` 抓包，过滤出其中的 ARP，显示的结果如图：
![arp 协议流程](https://ws4.sinaimg.cn/large/006tNc79gy1fi23aa7xbbj30xa03nmxo.jpg)

可以看到一共有两对 ARP 请求应答的过程，我们来逐个分析。

### ARP 请求报文

首先是机器 A 发送的 ARP 请求，详细的报文内容如下：
![ARP 请求报文](https://ws3.sinaimg.cn/large/006tNc79gy1fi23ccvypmj30ws0e6wgn.jpg)

报文列表的 `Info` 字段，对应的内容是：

> Who has 172.16.13.18? Tell 172.16.13.16

这是 wireshark 帮我们解析 ARP 报文，并用英语表达出来。这句话生动地概括了 ARP 请求的意思：**谁知道 172.16.13.18 的物理地址？告诉 172.16.13.16。**

这个以太网帧的源地址是 `50:7b:9d:ca:08:f0`，也就是机器 A 的 MAC 地址；目的地址是 `ff:ff:ff:ff:ff:ff`，这是一个以太网广播地址，**交换机**会把报文发送给网络中所有的主机。

以太网帧类型（Type）字段的值是 `0x0806`，表示该数据帧是个 ARP 请求或者应答。

帧的长度是 42，其中包括 14 字节以太网帧头部，以及 28 字节 ARP 请求数据。

以太网帧内部是 ARP 请求的具体内容，`Opcode` 为 1，表示这是 ARP 请求，发送端（也就是机器 A）的 IP 地址是 `172.16.13.16`，目的端（也就是机器 B）的 IP 地址是 `172.16.13.18`。发送端的 MAC 地址是填写的，而目的端 MAC 地址为空（全部为 0）。

### ARP 应答报文

第二个报文是 ARP 应答，详细报文内容如下图：
![ARP 应答报文](https://ws1.sinaimg.cn/large/006tNc79gy1fi23cw2798j30xl0en76n.jpg)

这个报文的 `info` 字段的信息是:

> 172.16.13.18 is at 50:7b:9d:e0:8b:a8

这也概括了 ARP 应答的意思：`172.16.13.18` 的 MAC 地址是 `50:7b:9d:e0:8b:a8`。

和上个报文不同，这是一个单播报文，直接发送到机器 A，帧的目的 MAC 地址是 `50:7b:9d:ca:08:f0`。因为机器 B 能够从接收到的 ARP 请求报文中获取机器 A 的 IP 地址和 MAC 地址，可以直接发送单播。

`Opcode` 字段的值为 2，表示这是个 ARP 应答报文，而且 ARP 报文中四个地址都填上了对应的值（因为此时机器已经知道通信双方的所有地址）。

另外一点需要注意的是，这个以太网帧的长度是 60 字节，因为它的以太网帧后面加了 18 字节的 padding（填充字符）。这是以太网帧的最小长度值，那为什么 ARP 请求报文的长度可以低于 60 呢？这和 wireshark 抓包的原理有关，wireshark 捕获的包并不是真正发送到线路上的帧，而是发送给网卡驱动的数据。因此，如果从机器 B 上抓包，会发现机器 B 接收到的这个 ARP 请求长度也是 60 字节。

通过两个报文的时间可以发现，ARP 地址解析的时间在 0.2ms 左右，说明这个网络并不是非常繁忙，速度还是挺快的。

### ARP cache validation 报文

后两个 ARP 报文也是一对：请求和应答，它们是从机器 B 发来查询机器 A MAC 地址的。

但是比较奇怪的是，ARP 请求报文以太网帧的目的地址居然不是广播，而是机器 A 的 MAC 地址？这和我们上面介绍的矛盾啊，为什么已经知道对方的 MAC 地址还要发送 ARP 报文去查询呢？

注意到，第三个报文距离上一个的时间间隔是 5s 左右，真正的 ICMP 报文（ping 应用传输的报文）已经在这个时间间隙中发送，也就是说机器 B 已经通过 A 发送的 ARP 报文把 ARP 保存到自己的缓存中了。第三个奇怪的 ARP 报文是为了**验证缓存的有效性**！

ARP 缓存都会有有效期，在 Linux 实现中，它会验证 ARP 缓存的有效性，并更新缓存记录的状态。因为已经知道对方的 ARP 记录，所以就没有必要再通过广播机制造成额外的网络资源浪费，这个 ARP 请求可以理解为： **请问 172.16.13.16，你的 MAC 地址还是 50:7b:9d:ca:08:f0吗？**。如果收到应答，说明该缓存记录有效；如果一直没有收到应答，则需要把缓存记录设置为失效或者删除，并在需要通信的时候使用正常的 ARP 请求获取对方的 MAC 地址。

我们可以在 `RFC1122` 的 `2.3.2.1` 小节（ARP Cache Validation）找到对应的文档说明：

> IMPLEMENTATION: Four mechanisms have been used, sometimes in
combination, to flush out-of-date cache entries.

> ......

> (2) Unicast Poll -- Actively poll the remote host by periodically sending a point-to-point ARP Request to it, and delete the entry if no ARP Reply is received from N successive polls. Again, the
timeout should be on the order of a minute, and typically N is 2.

**NOTE：**不同系统实现 ARP 缓存的机制不同，管理缓存有效期的方法也会有差异，并不能假设所有的系统都会使用上面的方法来验证缓存的有效性。

### 如果目的 IP 对应的机器不存在？

上面试验了正常通信情况下的 ARP 报文过程，如果目标地址不存在，ARP 协议又是怎么工作的呢？

我在自己的机器中做了实验，`ping` 一台不存在的主机，可以看到 `ping` 程序每隔一秒发送一个 `icmp` 报文，对应的结果是 `Destination Host Unreachable`:

```
➜  ~ ping 172.16.13.20
PING 172.16.13.20 (172.16.13.20) 56(84) bytes of data.
From 172.16.13.16 icmp_seq=1 Destination Host Unreachable
From 172.16.13.16 icmp_seq=2 Destination Host Unreachable
From 172.16.13.16 icmp_seq=3 Destination Host Unreachable
From 172.16.13.16 icmp_seq=4 Destination Host Unreachable
From 172.16.13.16 icmp_seq=5 Destination Host Unreachable
From 172.16.13.16 icmp_seq=6 Destination Host Unreachable
......
```

此时，使用 wireshark 抓包，发现网络上的 ARP 报文如下所示：

![向不存在的主机发送 ping 请求](https://ws2.sinaimg.cn/large/006tNc79gy1fi23dacq3qj30xi09htbb.jpg)

和 ICMP 报文类似，ARP 请求也会每秒都会发送，但是一直接收不到应答。

## Gratuitous ARP

除了标准的 ARP 之外，还有一种特殊的 ARP 报文，称为 Gratuitous ARP（免费 ARP）。这个报文也是广播报文，它的特殊性在于，它的报文中发送端 IP 地址和接收端 IP 地址都被设置为发送该报文的主机 IP。为什么要有这样一个特殊的报文呢？因为它有用，比如：

- 检测 IP 冲突。如果免费 ARP 请求接收到应答，说明当前网络上有另外一个和发送机器有相同 IP 的主机
- 可以用来更新网络中当前机器的 ARP 缓存。如果机器重新配置了 IP 地址，那么免费 ARP 报文能够把新的 IP-MAC 匹配关系广播到网络中，接收到报文的机器更新自己的 ARP 缓存记录，这样就不会有因为 ARP 缓存失效导致的网络问题

如果机器 A 重新配置了 IP 地址，那么 <IP-MAC> 的对应关系就发生了变化，网络中保存的旧 ARP 表项都失效，无法继续使用，会导致 ping 错误。Linux 系统中可以使用 `arping` 命令行来发送 Gratuitous ARP，让网络中所有主机更新当前机器的 ARP 记录：

```
➜  ~  arping -A -I eth0 172.16.42.161
```

这个命令就是把机器上 `eth0` 绑定的 MAC 地址和 `172.16.42.161` 作为 ARP 记录发送出去。

## ARP 的缺陷

ARP 协议很简单，通过缓存机制和 Gratuitous ARP 能够提供便利和高效的地址解析功能。尽管如此，和所有的网络协议一样，它并不是完美的，根据上面的解析，我们知道能发现它的两个不足之处：

1. ARP 报文没有任何认证，假设所有的机器都可靠而且诚信，所以 ARP 报文（尤其是应答）可以伪造。（窃听报文，或者报文转发）-> ARP spoofing
2. ARP 报文没有状态，机器可以在没有收到请求的时候直接发送应答。虽然特性能够有效地来更新 ARP 记录，也可能被恶意利用

ARP 报文另外一个让人迷惑的地方在于它在网络协议栈的位置：它既不属于二层（数据链路层），因为它不能把用户数据发送到目的机器，而且它其实是包在以太网帧里面；但它也不是三层（网络层），因为它没有在网络中寻路的功能。**它是介于二层和三层之间，更像是润滑剂的功能，帮助二层和三层正常工作。**

这是网络协议栈非常重要的一个特性，**网络协议是非常实用性的**。每个协议的产生为为了解决某个现实的问题，因为历史局限和现实设计问题，它可能并不完美，但是不妨碍它为我们提供服务。

## 参考资料

- [Networking Basics: How ARP Works](https://www.tummy.com/articles/networking-basics-how-arp-works/)
- [Address Resolution Protocol Tutorial, How ARP work, ARP Message Format](http://www.omnisecu.com/tcpip/address-resolution-protocol-arp.php)
- [The TCP/IP Guide: Address Resolution Protocol (ARP)](http://www.tcpipguide.com/free/t_TCPIPAddressResolutionProtocolARP.htm)
- [wireshark wiki: Address Resolution Protocol (ARP)](https://wiki.wireshark.org/AddressResolutionProtocol)
- [wireshark wiki: Gratuitous ARP](https://wiki.wireshark.org/Gratuitous_ARP)
- [ServerFault question: Why ARP request is unicast?](https://serverfault.com/questions/409865/why-arp-request-is-unicast)
- [Guide to IP Layer Network Administration with Linux: Address Resolution Protocol (ARP)](http://linux-ip.net/html/ether-arp.html)