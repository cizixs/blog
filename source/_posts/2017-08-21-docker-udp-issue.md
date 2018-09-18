---
layout: post
title: "docker 容器网络下 UDP 协议的一个问题"
excerpt: "最近在工作中遇到一个 docker 容器下 UDP 协议网络不通的问题，困扰了很久，也比较有意思，所以想写下来和大家分享。"
categories: blog
tags: [docker, linux, network, udp, socket, container]
comments: true
share: true
---

最近在工作中遇到一个 docker 容器下 UDP 协议网络不通的问题，困扰了很久，也比较有意思，所以想写下来和大家分享。

我们有个应用是 UDP 协议的，部署上去发现无法工作，但是换成 TCP 协议是可以的（应用同时支持 UDP、TCP 协议，切换成 TCP 模式发现一切正常）。虽然换成 TCP 能解决问题，但是我们还是想知道到底 UDP 协议在网络模式下为什么会出现这个问题，以防止后面其他 UDP 应用会有异常。

这个问题抽象出来是这样的：如果有 UDP 服务运行在主机上（或者运行在网络模型为 Host 的容器里），并且监听在 `0.0.0.0` 地址（也就是所有的 ip 地址），从运行在 docker bridge 网络的容器运行客户端访问服务，两者通信有问题。

注意以上的的限制条件，通过测试，我们发现下来几种情况都是正常的：

- 使用 TCP 协议没有这个问题，这个已经说过了
- 如果 UDP 服务器监听在 eth0 IP 地址上也不会出现这个问题
- 并不是所有的应用都有这个问题，我们的 DNS（dnsmasq + kubeDNS） 也是同样的部署方式，但是功能都正常

这个问题在 docker 上也有 issue 记录：https://github.com/moby/moby/issues/15127，但是目前并没有合理的解决方案。

这篇文章就分析一下出现这个问题的原因，希望给同样遇到这个问题的读者提供些帮助。

## 问题重现

这个问题很容易重现，我的实验是在 ubuntu16.04 下用 `netcat` 命令完成的，其他系统应该类似。在主机上通过 nc 监听 56789 端口，然后在容器里使用 `nc` 发数据。第一个报文是能发送出去的，但是以后的报文虽然在网络上能看到，但是对方无法接收。

在主机上运行 nc UDP 服务器（`-u` 表示 UDP 协议，`-l` 表示监听的端口）
```
$ nc -ul 56789
```

然后启动一个容器，运行客户端：

```
$ docker run -it apline sh
/ # nc -u 172.16.13.13 56789
```

nc 的通信是双方的，不管对方输入什么字符，回车后对方就能立即收到。但是在这个模式下，客户端第一次输入对方能够收到，后续的报文对方都收不到。

在这个实验中，容器使用的是 docker 的默认网络，容器的 ip 是 172.17.0.3，通过 veth pair（图中没有显示）连接到虚拟网桥 docker0（ip 地址为 172.17.0.1），主机本身的网络为 eth0，其 ip 地址为 172.16.13.13。

```
 172.17.0.3
+----------+
|   eth0   |
+----+-----+
     |
     |
     |
     |
+----+-----+          +----------+
| docker0  |          |  eth0    |
+----------+          +----------+
172.17.0.1            172.16.13.13

```

## tcpdump 抓包

遇到这种疑难杂症，第一个想到的抓包，我们需要在 docker0 上抓包，因为这是报文必经过的地方。通过过滤容器的 ip 地址，很容器找到感兴趣的报文：

```
$ tcpdump -i docker0 -nn host 172.17.0.3
```

为了模拟多数应用一问一答的通信方式，我们一共发送三个报文，并用 tcpdump 抓取 docker0 接口上的报文：

1. 客户端先向服务器端发送 `hello` 字符串
2. 服务器端回复 `world` 
3. 客户端继续发送 `hi` 消息

抓包的结果如下，可以发现第一个报文发送出去没有任何问题（因为 UDP 是没有 ACK 报文的，所以客户端无法知道对方有没有收到，这里说的没有问题是值没有对应的 ICMP 报文），但是第二个报文从服务端发送的报文，对方会返回一个 `ICMP` 告诉端口 38908 不可达；第三个报文从客户端发送的报文也是如此。以后的报文情况类似，双方再也无法进行通信了。

```
11:20:43.973286 IP 172.17.0.3.38908 > 172.16.13.13.56789: UDP, length 6
11:20:50.102018 IP 172.17.0.1.56789 > 172.17.0.3.38908: UDP, length 6
11:20:50.102129 IP 172.17.0.3 > 172.17.0.1: ICMP 172.17.0.3 udp port 38908 unreachable, length 42
11:20:54.503198 IP 172.17.0.3.38908 > 172.16.13.13.56789: UDP, length 3
11:20:54.503242 IP 172.16.13.13 > 172.17.0.3: ICMP 172.16.13.13 udp port 56789 unreachable, length 39
```

而此时主机上 UDP nc 服务器并没有退出，使用 `lsof -i :56789` 可能看到它仍然在监听着该端口。

## 问题原因

从网络报文的分析中可以看到服务端返回的报文源地址不是我们预想的 eth0 地址，而是 docker0 的地址，而客户端直接认为该报文是非法的，返回了 ICMP 的报文给对方。

那么问题的原因也可以分为两个部分：

1. 为什么应答报文源地址是**错误的**？
2. 既然 UDP 是无状态的，内核怎么判断源地址不正确呢？

### 主机多网络接口 UDP 源地址选择问题

第一个问题的关键词是：UDP 和多网络接口。因为如果主机上只有一个网络接口，发出去的报文源地址一定不会有错；而我们也测试过 TCP 协议是能够处理这个问题的。

通过搜索，发现这确实是个已知的问题。在 UNP（） 这本书中，已经描述过这个问题，下面是对应的内容：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1fiqmdohed0j31kw1je1kl.jpg)

这个问题可以归结为一句话：UDP 在多网卡的情况下，可能会发生服务器端源地址不对的情况，这是内核选路的结果。
为什么 UDP 和 TCP 有不同的选路逻辑呢？因为 UDP 是无状态的协议，内核不会保存连接双方的信息，因此每次发送的报文都认为是独立的，socket 层每次发送报文默认情况不会指明要使用的源地址，只是说明对方地址。因此，内核会为要发出去的报文选择一个 ip，这通常都是报文路由要经过的设备 ip 地址。

有了这个原因，还要解释一下问题：**为什么 dnsmasq 服务没有这个问题呢**？因此我使用 `strace` 工具抓取了 dnsmasq 和出问题应用的网络 socket 系统调用，来查看它们两个到底有什么区别。

dnsmasq 在启动阶段监听了 UDP 和 TCP 的 54 端口（因为是在本地机器上测试的，为了防止和本地 DNS 监听的 
DNS端口冲突，我选择了 54 而不是标准的 53 端口）：

```
socket(PF_INET, SOCK_DGRAM, IPPROTO_IP) = 4
setsockopt(4, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
bind(4, {sa_family=AF_INET, sin_port=htons(54), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
setsockopt(4, SOL_IP, IP_PKTINFO, [1], 4) = 0

socket(PF_INET, SOCK_STREAM, IPPROTO_IP) = 5
setsockopt(5, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
bind(5, {sa_family=AF_INET, sin_port=htons(54), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
listen(5, 5)                            = 0
```

比起 TCP，UDP 部分少了 `listen`，但是多个 `setsockopt(4, SOL_IP, IP_PKTINFO, [1], 4)` 这句。到底这两点和我们的问题是否有关，先暂时放着，继续看传输报文的部分。

dnsmasq 收包和发包的系统调用，直接使用 `recvmsg` 和 `sendmsg` 系统调用：

```
recvmsg(4, {msg_name(16)={sa_family=AF_INET, sin_port=htons(52072), sin_addr=inet_addr("10.111.59.4")}, msg_iov(1)=[{"\315\n\1 \0\1\0\0\0\0\0\1\fterminal19-0\5u5016\3"..., 4096}], msg_controllen=32, {cmsg_len=28, cmsg_level=SOL_IP, cmsg_type=, ...}, msg_flags=0}, 0) = 67

sendmsg(4, {msg_name(16)={sa_family=AF_INET, sin_port=htons(52072), sin_addr=inet_addr("10.111.59.4")}, msg_iov(1)=[{"\315\n\201\200\0\1\0\1\0\0\0\1\fterminal19-0\5u5016\3"..., 83}], msg_controllen=28, {cmsg_len=28, cmsg_level=SOL_IP, cmsg_type=, ...}, msg_flags=0}, 0) = 83
```

而出问题的应用 `strace` 结果如下：

```
[pid   477] socket(PF_INET6, SOCK_DGRAM, IPPROTO_IP) = 124
[pid   477] setsockopt(124, SOL_IPV6, IPV6_V6ONLY, [0], 4) = 0
[pid   477] setsockopt(124, SOL_IPV6, IPV6_MULTICAST_HOPS, [1], 4) = 0
[pid   477] bind(124, {sa_family=AF_INET6, sin6_port=htons(6088), inet_pton(AF_INET6, "::", &sin6_addr), sin6_flowinfo=0, sin6_scope_id=0}, 28) = 0

[pid   477] getsockname(124, {sa_family=AF_INET6, sin6_port=htons(6088), inet_pton(AF_INET6, "::", &sin6_addr), sin6_flowinfo=0, sin6_scope_id=0}, [28]) = 0
[pid   477] getsockname(124, {sa_family=AF_INET6, sin6_port=htons(6088), inet_pton(AF_INET6, "::", &sin6_addr), sin6_flowinfo=0, sin6_scope_id=0}, [28]) = 0

[pid   477] recvfrom(124, "j\201\2450\201\242\241\3\2\1\5\242\3\2\1\n\243\0160\f0\n\241\4\2\2\0\225\242\2\4\0"..., 2048, 0, {sa_family=AF_INET6, sin6_port=htons(38790), inet_pton(AF_INET6, "::ffff:172.17.0.3", &sin6_addr), sin6_flowinfo=0, sin6_scope_id=0}, [28]) = 168

[pid   477] sendto(124, "k\202\2\0210\202\2\r\240\3\2\1\5\241\3\2\1\v\243\5\33\3TDH\244\0220\20\240\3\2"..., 533, 0, {sa_family=AF_INET6, sin6_port=htons(38790), inet_pton(AF_INET6, "::ffff:172.17.0.3", &sin6_addr), sin6_flowinfo=0, sin6_scope_id=0}, 28) = 533
```

其对应的逻辑是这样的：使用 ipv6 绑定在 `0.0.0.0` 和 6088 端口，调用 `getsockname` 获取当前 socket 绑定的端口信息，数据传输过程使用的是 `recvfrom` 和 `sendto`。

对比下来，两者的不同有几点：

- 后者使用的是 ipv6，而前者是 ipv4
- 后者使用 `recvfrom` 和 `sendto` 传输数据，而前者是 `sendmsg` 和 `recvmsg`
- 前者有调用 `setsockopt` 设置 `IP_PKTINFO` 的值，而后者没有

因为是在传输数据的时候出错的，因此第一个疑点是 `sendmsg`  和 `sendto` 的某些区别导致选择源地址有不同，通过 `man sendto` 可以知道 `sendmsg` 包含了更多的控制信息在 `msghdr`。一个合理的猜测是 `msghdr` 中包含了内核选择源地址的信息！

通过查找，发现 `IP_PKTINFO` 这个选项就是让内核在 socket 中保存 IP 报文的信息，当然也包括了报文的源地址和目的地址。`IP_PKTINFO` 和 `msghdr` 的关系可以在这个 stackoverflow 中找到：https://stackoverflow.com/questions/3062205/setting-the-source-ip-for-a-udp-socket。


而 `man 7 ip` 文档中也说明了 `IP_PKTINFO` 是怎么控制源地址选择的：

```
IP_PKTINFO (since Linux 2.2)
              Pass  an  IP_PKTINFO  ancillary message that contains a pktinfo structure that supplies some information about the incoming packet.  This only works for datagram ori‐
              ented sockets.  The argument is a flag that tells the socket whether the IP_PKTINFO message should be passed or not.  The message itself can only be sent/retrieved as
              control message with a packet using recvmsg(2) or sendmsg(2).

                  struct in_pktinfo {
                      unsigned int   ipi_ifindex;  /* Interface index */
                      struct in_addr ipi_spec_dst; /* Local address */
                      struct in_addr ipi_addr;     /* Header Destination
                                                      address */
                  };

              ipi_ifindex  is the unique index of the interface the packet was received on.  ipi_spec_dst is the local address of the packet and ipi_addr is the destination address
              in the packet header.  If IP_PKTINFO is passed to sendmsg(2) and ipi_spec_dst is not zero, then it is used as the local source address for the  routing  table  lookup
              and  for  setting up IP source route options.  When ipi_ifindex is not zero, the primary local address of the interface specified by the index overwrites ipi_spec_dst
              for the routing table lookup.
```

如果 `ipi_spec_dst` 和 `ipi_ifindex` 不为空，它们都能作为源地址选择的依据，而不是让内核通过路由决定。

也就是说，通过设置 `IP_PKTINFO` socket 选项为 1，然后使用 `recvmsg` 和 `sendmsg` 传输数据就能保证源地址选择符合我们的期望。这也是 dnsmasq 使用的方案，而出问题的应用是因为使用了默认的 `recvfrom` 和 `sendto`。

### 关于 UDP 连接的疑惑

另外一个疑惑是：为什么内核会把源地址和之前不同的报文丢弃？认为它是非法的？因为我们前面已经说过，UDP 协议是无连接的，默认情况下 socket 也不会保存双方连接的信息。即使服务端发送报文的源地址有误，只要对方能正常接收并处理，也不会导致网络不通。

因为 conntrack，内核的 netfilter 模块会保存连接的状态，并作为防火墙设置的依据。它保存的 UDP 连接，只是简单记录了主机上本地 ip 和端口，和对端 ip 和端口，并不会保存更多的内容。

可以参考 intables info 网站的文章：http://www.iptables.info/en/connection-state.html#UDPCONNECTIONS。

在找到根源之前，我们曾经尝试过用 SNAT 来修改服务端应答报文的源地址，期望能够修复该问题。但是却发现这种方法行不通，为什么呢？

因为 SNAT 是在 netfilter 最后做的，在之前 netfilter 的 conntrack 因为不认识该 connection，直接丢弃了，所以即使添加了 SNAT 也是无法工作的。

那能不能把 conntrack 功能去掉呢？比如解决方案：

```
iptables -I OUTPUT -t raw -p udp --sport 5060 -j CT --notrack
iptables -I PREROUTING -t raw -p udp --dport 5060 -j CT --notrack
```

答案也是否定的，因为 NAT 需要 conntrack 来做翻译工作，如果去掉 conntrack 等于 SNAT 完全没用。


## 解决方案

知道了问题的原因，解决方案也就很容易找到。

### 使用 TCP 协议

如果服务端和客户端使用 TCP 协议进行通信，它们之间的网络是正常的。

```
$ nc -l 56789
```

### 监听在特定端口

使用 nc 启动一个 udp 服务器，监听在 eth0 上：

```
➜  ~ nc -ul 172.16.13.13 56789
```

`nc` 可以跟两个参数，分别代表 ip 和 端口，表示服务端监听在某个特定 ip 上。如果接收到的报文目的地址不是 172.16.13.13，也会被内核直接丢弃。

这种情况下，服务端和客户端也能正常通信。

### 改动应用程序实现

修改应用程序的逻辑，在 UDP socket 上设置 `IP_PKTIFO`，并通过 `recvmsg` 和 `sendmsg` 函数传输数据。

## 参考资料

- [IPv6:Which Source Address is used when you have many IPv6 addresses ? Default Address Selection](http://etherealmind.com/ipv6-which-address-multiple-ipv6-address-default-address-selection/)
- [powerDNS blog: on binding datagram udp sockets to the any address](https://blog.powerdns.com/2012/10/08/on-binding-datagram-udp-sockets-to-the-any-addresses/)
- [RFC3542: IPV6 source address selection](https://www.ietf.org/rfc/rfc3542.txt)
- [linux udp socket sendto operation not permitted](https://stackoverflow.com/questions/23859164/linux-udp-socket-sendto-operation-not-permitted)