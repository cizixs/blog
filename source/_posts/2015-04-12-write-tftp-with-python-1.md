---
layout: post
title: "编写自己的 tftp 客户端（1）"
excerpt: "使用 python 实现自己的 tftp 协议"
categories: 程序技术
tags: [python, tftp, socket]
comments: true
share: true
---


tftp 协议一般用来 PXE 协议中传输文件，因为协议内容比较简单，所以非常容易实现。关于 tftp 的详细信息，可以参考维基百科或者 RFC 1350。

这是两篇文章的第一篇，主要是 tftp 理论知识，第二篇是 python 实现的技术和细节还有具体程序的说明。

下面我会简单介绍 tftp 协议的主要内容，为后面编写 tftp client 和 server 做好准备。

## tftp 发送流程

1. client 端向 server 端 69 端口发送 RRQ（读请求）或者 WRQ（写请求），内容包括文件名、传输模式以及 RFC 2347 文档里补充的协商信息。
2. 如果有选项信息（协商信息），server 就先回复 option ACK；否则如果是 RRQ，就直接回复 DATA 数据报；如果是 WRQ，就回复 ACK。
3. 发送端会顺序地发送数据报。除了最后一个，每个数据包都是完整的 block size。接收方对收到的每一个报文发送应答（ACK）报文。
4. 发送数据小于 block size 的报文标志着通信的结束，如果数据大小恰好是 block size 的整数倍。发送方必须在最后发送一个数据大小是 0 的报文
5. 发送方每发送一个数据报文，只有接受到应答报文才会发送下一个数据报
6. 如果发送方在一定的时间没有接收到 ACK 报文，就必须重发数据报。


## tftp 数据报文格式

tftp 一共有五种报文：

1. RRQ
2. WRQ
3. Data
4. ACK
5. Error

前面的序号也是它们的 Opcode。

### RRQ 和 WRQ

RRQ 和 WRQ 的格式如下所示，其中 Opcode 为 1 代表 RRQ，Opcode 为 2 代表 WRQ。

            2 bytes     string    1 byte     string   1 byte
            ------------------------------------------------
           | Opcode |  Filename  |   0  |    Mode    |   0  |
            ------------------------------------------------

                       RRQ/WRQ packet


其中文件名 filename 是不定长的字符串，后面有一个比特的 0 作为分隔符。mode 是 `netascii`、`octet` 或者 `mail` 之一，大小写不敏感（octet 和 OcTEt 都可以接受）。

+ netascii：ascii 码传输，需要发送方和接收方自己解码 ascii 码
+ octet： 文件传输，字节模式
+ mail： 邮件传输

我们只考虑文件传输，其他两种模式的详细解释可以参看 [RFC 1350](https://tools.ietf.org/html/rfc1350)。

### Data 报文

                   2 bytes     2 bytes      n bytes
                   ----------------------------------
                  | Opcode |   Block #  |   Data     |
                   ----------------------------------

                        
上面是数据报文的格式，`Opcode` 是 3，`Block` 是数据报文的序号，因为只有 2 个字节，所以是 1-65535。当 `data block` 是 512 byte时，能接受的文件大小是 512 * 65536 = 32MB，不过后面的 tftp 修订允许 `data block` 达到 1468 byte（以太网 MTU（1500）- TFTP header（4） - UDP header （8）- IP header（20）），那么能够传输的大小是 1468 * 65535 = 93M。不过如果 tftp 实现循环利用序列号的话（65535 的下一个序号是 0），理论上可以传输的文件大小没有限制。

当传输的 data 大小 `n < block size` 的时候，就表明传输已经结束了。

### ACK 报文

                         2 bytes     2 bytes
                         ---------------------
                        | Opcode |   Block #  |
                         ---------------------

上面的 ACK 报文，`Opcode` 是 4， `Block` 应答的数据报序列号 ，WRQ 的 ACK 报文 Block 序列号是 0。

所有的报文都会有 ACK 报文作为应答，除了

1. 重复的 ACK
2. 结束报文
3. 报文传送超时


### 错误报文

               2 bytes     2 bytes      string    1 byte
               -----------------------------------------
              | Opcode |  ErrorCode |   ErrMsg   |   0  |
               -----------------------------------------

错误报文的格式如上所示，Opcode 是 5，ErrorCode 是 错误代码，ErrMsg 是错误信息，结尾以一个比特的 0 作为分隔符。

## 异常处理

### 传输终结
像前面说过的那样，如果接收端收到的数据长度小于 block size，说明传输完成需要终结整个过程。

这里的问题在于是否要对最后一个数据报文发送 ACK 应答？
答案是肯定的，对于接收端来说这个意义不大，比较数据已经完全接受，后面也没有事情要处理，直接做一下清理工作，关闭程序，就可以收工了。问题在于发送端，它发送了最后一个数据报文，但是不知道对方接收没有（因为底层使用的是 UDP 协议，你懂得）。如果没有收到最后一个数据报的 ACK，可能就不会终结处理程序（取决于具体的实现），造成想不到的错误。

好，现在接收端发送最后一个 ACK 报文，结束了吗？没有，它还要等待一段时间（默认的超时时间），如果还接收到发送端发送的最后一个数据报文，说明发送方没有收到 ACK。接收方必须重新发送 ACK，直到在等待时间里没有收到数据报文，或者超过默认的最大重发次数。
    
接收端在收到最后一个 ACK 报文的时候才知道整个传输过程结束了。


上面说明的情况是传输正常结束，如果在发送过程中出现错误或者请求无法被满足，就会发送 ERROR （opcode 为 5）的报文。接收到 ERROR 报文，一般情况下是要终止程序（如果能够处理的错误可能在处理完成后继续）。需要注意的是，ERROR 报文是发完就不管的，不像最后一个 ACK 报文，没有重发机制。所以需要接收方借助超时机制来检测到错误。

  
错误代码：

   Value   |   Meaning
--------   | ------------
   0       |  Not defined, see error message (if any).
   1       |  File not found.
   2       |  Access violation.
   3       |  Disk full or allocation exceeded.
   4       |  Illegal TFTP operation.
   5       |  Unknown transfer ID.
   6       |  File already exists.
   7       |  No such user.
   
### 数据丢失和超时
tftp 没有提供报文的错误检查机制，接收方无法知道自己接收的数据是不是被篡改过或者传输过程中有错误，只能认为所有接收的报文都是正确的（当然如果报文格式就错了，就是另外一回事了）。为了保证所有的数据都能被接收，每一个报文都必须要确认（ACK）。

数据丢失、对端主机服务无法访问这些错误需要借助超时机制来检测，在每次发送一个报文的时候，要设置一个计时器，如果在计时器超时的时候还没有接收到下一个数据，就要重发上一个报文；如果超时次数超过最大值，就要终止程序。**当接收到新的报文时，不管这个报文是正确的还是重复的，都要重置超市计数器为0。**


## 参考文档

+ [RFC 1350: tftp revision](https://tools.ietf.org/html/rfc1350)
+ [RFC 2347: tftp option extension](https://tools.ietf.org/html/rfc2347)
+ [RFC 2348: tftp Blocksize Option](http://www.faqs.org/rfcs/rfc2348.html)
+ [RFC 2349: TFTP timeout interval and transfer size option](http://www.faqs.org/rfcs/rfc2349.html)
+ [wikipedia TFTP protocol](https://en.wikipedia.org/wiki/Trivial_File_Transfer_Protocol)
