---
layout: post
title: "使用 tc netem 模拟网络异常"
excerpt: "在某些情况下，我们需要模拟网络很差的状态来测试软件能够正常工作，比如网络延迟、丢包、乱序、重复等。linux 系统强大的流量控制工具 tc 能很轻松地完成。"
categories: blog
tags: [linux, network, tc]
comments: true
share: true
---

在某些情况下，我们需要模拟网络很差的状态来测试软件能够正常工作，比如网络延迟、丢包、乱序、重复等。linux 系统强大的流量控制工具 tc 能很轻松地完成，tc 命令行是 `iproute2` 软件包中的软件，可以根据系统版本自行安装。

流量控制是个系统而复杂的话题，tc 能做的事情很多，除了本文介绍的还有带宽控制、优先级控制等等，这些功能是通过类似的**模块**组件实现的，这篇文章介绍的功能主要是通过 `netem` 这个组件实现的。`netem` 是 `Network Emulator` 的缩写，关于更多功能以及参数的详细解释可以参阅 `tc-netem` 的 man page。

## 网络状况模拟

网络状况欠佳从用户角度来说就是下载东西慢（网页一直加载、视频卡顿、图片加载很久等），从网络报文角度来看却有很多情况：延迟（某个机器发送报文很慢）、丢包（发送的报文在网络中丢失需要一直重传）、乱序（报文顺序错乱，需要大量计算时间来重新排序）、重复（报文有大量重复，导致网络拥堵）、错误（接收到的报文有误只能丢弃重传）等。

对于这些情况，都可以用 netem 来模拟。需要注意的是，netem 是直接添加到网卡上的，也就是说所有从网卡发送出去的包都会收到配置参数的影响，所以最好搭建临时的虚拟机进行测试。

在下面的例子中 `add` 表示为网卡添加 netem 配置，`change` 表示修改已经存在的 netem 配置到新的值，如果要删除网卡上的配置可以使用 `del`：

```
# tc qdisc del dev eth0 root
```

### 1. 模拟延迟传输

最简单的例子是所有的报文延迟 100ms 发送：
```
# tc qdisc add dev eth0 root netem delay 100ms
```

如果你想在一个局域网里模拟远距离传输的延迟可以用这个方法，比如实际用户会访问外国网站，延迟为 120ms，而你测试环境网络交互只需要 10ms，那么只要添加 110 ms 额外延迟就行。

在我本地的虚拟机中实验结果：

```
[root@node02 ~]# tc qdisc replace dev enp0s8 root netem delay 100ms
[root@node02 ~]# ping 172.17.8.100
PING 172.17.8.100 (172.17.8.100) 56(84) bytes of data.
64 bytes from 172.17.8.100: icmp_seq=1 ttl=64 time=101 ms
64 bytes from 172.17.8.100: icmp_seq=2 ttl=64 time=100 ms
64 bytes from 172.17.8.100: icmp_seq=3 ttl=64 time=102 ms
^C
--- 172.17.8.100 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 100.725/101.370/102.048/0.653 ms
```

如果在网络中看到非常稳定的时延，很可能是某个地方加了定时器，因为网络线路很复杂，传输过程一定会有变化。因此实际情况网络延迟一定会有变化的，`netem` 也考虑到这一点，提供了额外的参数来控制延迟的时间分布，完整的参数列表为：

```
DELAY := delay TIME [ JITTER [ CORRELATION ]]]
    [ distribution { uniform | normal | pareto |  paretonormal } ]
```

除了延迟时间 `TIME` 之外，还有三个可选参数：

- `JITTER`：抖动，增加一个随机时间长度，让延迟时间出现在某个范围
- `CORRELATION`：相关，下一个报文延迟时间和上一个报文的相关系数
- `distribution`：分布，延迟的分布模式，可以选择的值有 `uniform`、`normal`、`pareto` 和 `paretonormal`


先说说 `JITTER`，如果设置为 `20ms`，那么报文延迟的时间在 100ms  ± 20ms 之间（90ms - 110ms），具体值随机选择：

```
[root@node02 ~]# tc qdisc replace dev enp0s8 root netem delay 100ms 20ms
[root@node02 ~]# ping 172.17.8.100
PING 172.17.8.100 (172.17.8.100) 56(84) bytes of data.
64 bytes from 172.17.8.100: icmp_seq=1 ttl=64 time=112 ms
64 bytes from 172.17.8.100: icmp_seq=2 ttl=64 time=89.7 ms
64 bytes from 172.17.8.100: icmp_seq=3 ttl=64 time=114 ms
......
```

`CORRELATION` 指相关性，因为网络状况是平滑变化的，短时间里相邻报文的延迟应该是近似的而不是完全随机的。这个值是个百分比，如果为 `100%`，就退化到固定延迟的情况；如果是 `0%` 则退化到随机延迟的情况

```
[root@node02 ~]# tc qdisc replace dev enp0s8 root netem delay 100ms 20ms 50%
[root@node02 ~]# ping 172.17.8.100
PING 172.17.8.100 (172.17.8.100) 56(84) bytes of data.
64 bytes from 172.17.8.100: icmp_seq=1 ttl=64 time=116 ms
64 bytes from 172.17.8.100: icmp_seq=2 ttl=64 time=89.7 ms
64 bytes from 172.17.8.100: icmp_seq=3 ttl=64 time=90.8 ms
64 bytes from 172.17.8.100: icmp_seq=4 ttl=64 time=96.4 ms
64 bytes from 172.17.8.100: icmp_seq=5 ttl=64 time=90.5 ms
```

报文的分布和很多现实事件一样都满足某种统计规律，比如最常用的正态分布。因此为了更逼近现实情况，可以使用 `distribution` 参数来限制它的延迟分布模型。比如让报文延迟时间满足正态分布：

```
# tc qdisc change dev eth0 root netem delay 100ms 20ms distribution normal
[root@node02 ~]# tc qdisc replace dev enp0s8 root netem delay 100ms 20ms distribution normal
[root@node02 ~]# ping 172.17.8.100
PING 172.17.8.100 (172.17.8.100) 56(84) bytes of data.
64 bytes from 172.17.8.100: icmp_seq=1 ttl=64 time=119 ms
64 bytes from 172.17.8.100: icmp_seq=2 ttl=64 time=102 ms
64 bytes from 172.17.8.100: icmp_seq=3 ttl=64 time=115 ms
64 bytes from 172.17.8.100: icmp_seq=4 ttl=64 time=105 ms
64 bytes from 172.17.8.100: icmp_seq=5 ttl=64 time=119 ms

```

这样的话，大部分的延迟会在平均值的一定范围内，而很少接近出现最大值和最小值的延迟。

其他分布方法包括：[uniform](https://en.wikipedia.org/wiki/Uniform_distribution_(continuous))、[pareto](https://en.wikipedia.org/wiki/Pareto_distribution) 和 `paretonormal`，这些分布我没有深入去看它们的意思，感兴趣的读者可以自行了解。

对于大多数情况，随机在某个时间范围里延迟就能满足需求的。

### 2. 模拟丢包率

另一个常见的网络异常是因为丢包，丢包会导致重传，从而增加网络链路的流量和延迟。netem 的 `loss` 参数可以模拟丢包率，比如发送的报文有 50% 的丢包率（为了容易用 ping 看出来，所以这个数字我选的很大，实际情况丢包率可能比这个小很多，比如 `0.5%`）：

```
[root@node02 ~]# tc qdisc change dev enp0s8 root netem loss 50%
[root@node02 ~]# ping 172.17.8.100
PING 172.17.8.100 (172.17.8.100) 56(84) bytes of data.
64 bytes from 172.17.8.100: icmp_seq=1 ttl=64 time=0.716 ms
64 bytes from 172.17.8.100: icmp_seq=3 ttl=64 time=0.713 ms
64 bytes from 172.17.8.100: icmp_seq=5 ttl=64 time=0.719 ms
64 bytes from 172.17.8.100: icmp_seq=7 ttl=64 time=0.938 ms
64 bytes from 172.17.8.100: icmp_seq=10 ttl=64 time=0.594 ms
64 bytes from 172.17.8.100: icmp_seq=11 ttl=64 time=0.698 ms
64 bytes from 172.17.8.100: icmp_seq=12 ttl=64 time=0.681 ms
```

可以从 `icmp_seq` 序号看出来大约有一半的报文丢掉了，和延迟类似，丢包率也可以增加一个相关系数，表示后一个报文丢包概率和它前一个报文的相关性：

```
# tc qdisc change dev eth0 root netem loss 0.3% 25%
```

这个命令表示，丢包率是 0.3%，并且当前报文丢弃的可能性和前一个报文 25% 相关。默认的丢包模型为随机，loss 也支持 `state`（4-state Markov 模型） 和 `gemodel`（Gilbert-Elliot 丢包模型） 两种模型的丢包，因为两者都相对负责，这里也不再介绍了。

需要注意的是，丢包信息会发送到上层协议，如果是 TCP 协议，那么 TCP 会进行重传，所以对应用来说看不到丢包。这时候要模拟丢包，需要把 loss 配置到王桥或者路由设备上。

### 3. 模拟包重复

报文重复和丢包的参数类似，就是重复率和相关性两个参数，比如随机产生 50% 重复的包：
```
[root@node02 ~]# tc qdisc change dev enp0s8 root netem duplicate 50%
[root@node02 ~]# ping 172.17.8.100
PING 172.17.8.100 (172.17.8.100) 56(84) bytes of data.
64 bytes from 172.17.8.100: icmp_seq=1 ttl=64 time=0.705 ms
64 bytes from 172.17.8.100: icmp_seq=1 ttl=64 time=1.03 ms (DUP!)
64 bytes from 172.17.8.100: icmp_seq=2 ttl=64 time=0.710 ms
......
```

### 4. 模拟包损坏

报文损坏和报文重复的参数也类似，比如随机产生 2% 损坏的报文（在报文的随机位置造成一个比特的错误）：
```
# tc qdisc add dev eth0 root netem corrupt 2%
```

### 5. 模拟包乱序

网络传输并不能保证顺序，传输层 TCP 会对报文进行重组保证顺序，所以报文乱序对应用的影响比上面的几种问题要下。

报文乱序可前面的参数不太一样，因为上面的报文问题都是独立的，针对单个报文做操作就行，而乱序则牵涉到多个报文的重组。模拟报乱序一定会用到延迟（因为模拟乱序的本质就是把一些包延迟发送），netem 有两种方法可以做。第一种是固定的每隔一定数量的报文就乱序一次：

每 5 个报文（第 5、10、15…报文）会正常发送，其他的报文延迟 100ms：
```
# tc qdisc change dev enp0s8 root netem reorder 50% gap 3 delay 100ms
[root@node02 ~]# ping -i 0.05 172.17.8.100
PING 172.17.8.100 (172.17.8.100) 56(84) bytes of data.
64 bytes from 172.17.8.100: icmp_seq=1 ttl=64 time=0.634 ms
64 bytes from 172.17.8.100: icmp_seq=4 ttl=64 time=0.765 ms
64 bytes from 172.17.8.100: icmp_seq=2 ttl=64 time=102 ms
64 bytes from 172.17.8.100: icmp_seq=3 ttl=64 time=100 ms
64 bytes from 172.17.8.100: icmp_seq=5 ttl=64 time=100 ms
64 bytes from 172.17.8.100: icmp_seq=7 ttl=64 time=50.3 ms
64 bytes from 172.17.8.100: icmp_seq=6 ttl=64 time=100 ms
64 bytes from 172.17.8.100: icmp_seq=8 ttl=64 time=100 ms
......
```

要想看到 ping 报文的乱序，我们要保证发送报文的间隔小于报文的延迟时间 `100ms`，这里用 `-i 0.05` 把发送间隔设置为 `50ms`。

第二种方法的乱序是相对随机的，使用概率来选择乱序的报文：
```
[root@node02 ~]# tc qdisc change dev enp0s8 root netem reorder 50% 15% delay 300ms
[root@node02 ~]# ping -i 0.05 172.17.8.100
PING 172.17.8.100 (172.17.8.100) 56(84) bytes of data.
64 bytes from 172.17.8.100: icmp_seq=1 ttl=64 time=0.545 ms
64 bytes from 172.17.8.100: icmp_seq=5 ttl=64 time=120 ms
64 bytes from 172.17.8.100: icmp_seq=2 ttl=64 time=300 ms
64 bytes from 172.17.8.100: icmp_seq=8 ttl=64 time=19.8 ms
64 bytes from 172.17.8.100: icmp_seq=3 ttl=64 time=301 ms
64 bytes from 172.17.8.100: icmp_seq=9 ttl=64 time=28.3 ms
64 bytes from 172.17.8.100: icmp_seq=4 ttl=64 time=300 ms
64 bytes from 172.17.8.100: icmp_seq=11 ttl=64 time=35.5 ms
......
```

50% 的报文会正常发送，其他报文（1-50%）延迟 300ms 发送，这里选择的延迟很大是为了能够明显看出来乱序的结果。

## 两个工具

netem 在 tc 中算是比较简单的模块，如果要实现流量控制或者精细化的过滤需要更复杂的配置。这里推荐两个小工具，它们共同的特点是用法简单，能满足特定的需求，而不用自己去倒腾 tc 的命令。

### wondershaper

netem 只能模拟网络状况，不能控制带宽，[wondershaper](https://www.hecticgeek.com/2012/02/simple-traffic-shaping-ubuntu-linux/) 能完美解决这个问题。wondershaper 的使用非常简单，只有三个参数：网卡名、下行限速、上行限速。比如要设置网卡下载速度为 200kb/s，上传速度为 `150kb/s`：

```
wondershaper enp0s8 200 150
```

### comcast

[comcast](https://github.com/tylertreat/comcast) 是一个跨平台的网络模拟工具，旨在其他平台（OSX、Windows、BSD）也提供类似网络模拟的功能。

它的使用也相对简单：

```
$ comcast --device=eth0 --latency=250 \
    --target-bw=1000 --default-bw=1000000 \
    --packet-loss=10% \
    --target-addr=8.8.8.8,10.0.0.0/24 \
    --target-proto=tcp,udp,icmp \
    --target-port=80,22,1000:2000
```

-  `--device` 说明要控制的网卡为 `eth0`
- `--latency` 指定 250ms 的延迟
- `--target-bw`指定目标带宽
- `--default-bw` 指定默认带宽
- `--packet-loss` 是丢包率
- `--target-addr`、`--target-proto`、`--target-port` 参数指定在满足这些条件的报文上实施上面的配置

## 总结

可以看出，tc 的 netem 模块主要用来模拟各种网络的异常状况，本身并没有提供宽带限制的功能，而且一旦在网卡上配置了 netem，该网卡上所有的报文都会受影响，如果想精细地控制部分报文，需要用到 tc 的 [filter](http://lartc.org/howto/lartc.qdisc.filters.html) 功能。

## 参考资料

- [the Linux Foundation wiki: netem](https://wiki.linuxfoundation.org/networking/netem)
- [Definition of a general and intuitive loss model for packet networks and its implementation in the Netem module in the Linux kernel](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.172.7072&rep=rep1&type=pdf)