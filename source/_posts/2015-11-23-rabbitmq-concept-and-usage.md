---
layout: post
title: "python 和消息机制（二）：Rabbitmq 概念和使用"
excerpt: "Rabbitmq 是常用的消息队列，因为概念比较多，理解起来会有点困难。这篇文章希望能把常用的概念说清楚，这样在使用的时候才能减少出错的几率，也会加深对代码的掌控。"
categories: blog
tags: [rabbitmq, queue, message, distribute]
comments: true
share: true
---


## rabbitmq 简介

Rabbitmq 是 amqp 的 Erlang 实现，也是现在非常流行的一种消息机制。在前面一片文章，我们也提到了消息机制的几个优点：

+ 异步（asynchronous）：耗时的工作可以直接丢给消费者，不会阻塞生产者
+ 可扩展（scale）：消息机制的工作模式，在理论上可以让无限的消费者接入进来，使得横向扩展变得异常简单
+ 模块化（modulize）：生产者和消费者不需要知道双方的存在，而且可以使用不同的语言和框架，在物理上位于不同的地方。

消息机制当然也有不便之处，比如分布式的架构使得测试和调试变得复杂；因为消费者无法保证能收到任务并完成，并且无法通知生产者（后面会详细讲解这一点），也对很多场景带来麻烦。


## rabbitmq 概念和架构

![](http://ww1.sinaimg.cn/large/728b3d6djw1exeozui9f8j20sg0lcdk6.jpg)

在生产者-消费者模型中，rabbitmq 充当着 broker 的角色。在 rabbitmq 的内部，还有以下的概念：

+ **Exchanges**：生产者把消息发送到某个 exchange，exchange 的主要工具就是根据一定的规则把消息分发到不同的 queues

+ **Queues**：消息最终被发送到的地方，消费者也是从这里拿消息进行处理

+ **Routing key**：queue 要绑定（binding）到某个 exchange 才有可能接受这个 exchange 转发过来的消息，它们之间的绑定要有 binding key（当 exchange 是 fanout 类型的时候并不需要）。然后每个消息发过来的时候，都会带着 routing key，根据 exchange 的类型，binding key 和 routing key，消息才能正确被转发。

+ **Connection**：生产者和消费者需要连接到 rabbitmq 才能发送和读取消息，这个连接就是 connection，本质上就是 TCP 连接。

+ **Channel**：为了减少网络负载，和减少 TCP 链接数。多个不同的 生产者可以在同一个 TCP 发送消息，只不过在这个 connection 下面独自的 channel 里。你可以把 channel 想象成一根网线里独立的细线。每个 channel 都有自己的  id，用来标识自己。虽然有 connection，不过所有的通信都是在对应的 channel 里进行的。

+ **vhost**：Virtual host，是起到隔离作用的。每一个 vhost 都有自己的 exchanges 和 queues，它们互不影响。不同的应用可以跑在相同的 rabbitmq 上，使用 vhost 把它们隔离开就行。默认情况下，rabbitmq 安装后，默认的 vhost 是 `/`。 

### 什么时候定义 queue？

queue 需要显式地定义，每个 queue 都有一个名字来唯一标示。 生产者和消费者都可以通过 `queue.declare` 命令来定义 queue，除了命令 queue 还有其他可以配置的属性：

+ exclusive：如果设置成 true，那么只有定义 queue 的应用可以使用
+ auto-delete：如果设置成 true，当最后一个消费者断开监听的时候，就会自动删除。结合 exclusive 使用，可以建立一个临时的 queue。

如果你要定义一个已经存在的 queue，如果两者的名字和属性都相同，rabbitmq 什么都不会做；而如果两者属性不同，就会返回失败。如果你只想检查某个 queue 是否存在，可以把 `queue.declare` 的 `passive` 设置成 true，这样当 queue 存在的时候就会返回成功，而 queue 不存在的时候也没有去创建，只是返回失败。

在消费者监听某个 queue 之前，要确保它一定存在，所以消费者要提前 declare 这个 queue。
如果生产者发送的消息，没有 queue 可以转发，就会被丢掉。所以生产者在发消息之前也要 declare 这个 queue，除非消息被丢掉是预期的行为。

### 绑定和转发

上图中很重要的部分就是 exchange 和 queue 的关联，它决定了消息机制在 rabbitmq 内部怎么转发的。生产者把消息发送到 exchange，然后消息根据一定的规则会被转发到 queues，消费者从 queues 获取消息。

决定消息转发规则的是 exchange 的类型 和 routing key。

queue 和 exchange 也是通过 routing key 绑定的，不过为了不混淆，我们在这篇文章称之为 binding key。每条消息也都有 routing key，当然可能为空，当消息到达 exchange 的时候， rabbitmq 会匹配消息的 routing key 和 binding key，如果匹配，就会转发到对应的 queue，如果不匹配，就会丢弃消息。

下面就看看每种不同的 exchange 类型下面，routing key 是怎么决定转发的：

1. exchange type：direct
    这种比较简单，当 routing key 和 binding key 相同时，消息就会转发到绑定的 queue。需要注意的是：如果有两个 queue 绑定到一个 exchange，并且 binding key 一样，那么消息会发到两个 queue。默认情况下，rabbitmq 会有 name 为空的 exchange，类型为 direct，当 declare 一个 queue 的时候，默认会绑定到这个 exchange，binding key 是 queue name。

2. exchange type：fanout
    当 exchange 类型是 fanout 的时候，并不需要 binding key，发送到该 exchange 的消息会自动转发给所有绑定的 queues ，也就是说每个 queue 都会收到一份消息。

3. exchange type：topic
    这种类型的 exchange 更复杂，也更灵活，能实现更多有趣的功能。routing key 和原来一样，不过 binding key 的名字中可以使用三种特殊的符号：`.`、`*`、`#`。`.` 把 binding key 分割成不同的单词（word），`*`  匹配一个单词，`#` 可以匹配 0 个或者任意多个单词。
举个例子说明一下，可能会清楚一些。如果有一个处理用户多媒体的应用，那么可以发送所有的消息到 `media` exchange 中，然后 queues 通过 binding key `video.upload` 和 `image.delete` 绑定到该 exchange，接受视频上传和图片删除的消息，这和 direct 类型没有区别。后面就有趣了，`video.*` 接受所有视频有关的消息，`*.delete` 接受所有删除操作的消息，`#` 接受所有的消息。

### 消费者如何获取消息？

消费者只要建立到某个 rabbitmq 的连接，然后获取到 channel，就可以使用 AMQP 提供的 `basic.consume` 命令就能获取到某个 queue 的消息。`basic.consume` 会让消费者一直处于 loop，只要监听的 queue 里有消息，就会取出来来处理。

如果只想读取一条消息，可以使用 `basic.get`。不要在 `while` 循环中使用 `basic.get` 来不断获取消息，而是要直接使用 `basic.consume`。因为 `basic.get` 对 rabbitmq 的压力很大，需要每次 subscribe 到 queue，然后 unsubscribe。

如果发送到 queue 的消息，没有消费者来处理，就会一直等待。

还有一个问题需要考虑：如果多个消费者同时监听一个 queue，那么里面的消息会怎么分配呢？默认情况下，rabbitmq 会采用 round-robin 的方式，也就是轮流的机制。如果有三个消费者同时监听已过期 queue，rabbitmq 会把过来的消息一次分发给它们，当然每条消息只会发给一个消费者。

消费者处理了消息，还要向 rabbitmq 发送确认（ack）。确认有两种方式：要么消费者在获取消息之后，显示地发送 `basic.ack` 消息给 rabbitmq；要么在监听一个 queue 的时候，设置 `auto_ack` 参数，这样每次消费者收到消息，rabbitmq 就自动认为这个消息已经 ack 了。确认消息的目的是告诉 rabbitmq，这条消息已经处理，可以把它从 queue 里面删除。那么这里还有一个问题：如果没有设置 `auto_ack` 参数，消费者也没有发送 `basic.ack` 回去，rabbitmq 会认为这个消息没有被正确处理，会再次发送给其他消费者，同时会把没有 ack 的消费者做下标记，之后不会在发送消息过来。

上面都是消费者要处理消息，如果消费者不愿意处理这个消息呢？只要消息还没有被 ack（如果消息被 ack 了就没有办法啦），就有两个办法可以做到：

1. 让消费者从 rabbitmq 断开连接，这时候 rabbitmq 会认为消息没有正确处理，然后交给其他消费者来做
2. 在 rabbitmq 2.0.0 以后，也可以使用 `basic.reject` 命令来拒绝一条消息。

当然如果你确定这个消息，没有办法被其他消费者处理，可以直接 ack，然后不去处理就好。

## rabbitmq 权限管理

理解权限管理之前，先要看一下 rabbitmq 的用户管理。

+ 添加用户：`rabbitmqctl add_user username password`
+ 删除用户：`rabbitmqctl delete_user username`
+ 列出所有用户：`rabbitmqctl list_users`
+ 修改某个用户的密码：`rabbitmqctl change_password username new_password`

每个用户控制 vhost ，单个用户可以有管理多个 vhost，每个 vhost 的权限都可以单独设置。权限一共有三种：read，write 和 config。

+ read：和消息消费有关的操作，也包括 exchange 的绑定
+ write：和消息发布有关的操作，也包括 queue 的绑定
+ configure：queue 和 exchange 的创建和删除

rabbitmqctl set_permissions 命令可以设置某个用户的权限，更详细的内容可以参考[官方文档](https://www.rabbitmq.com/access-control.html)。

## 参考资料

+ [Rabbitmq in Action](http://www.amazon.com/RabbitMQ-Action-Distributed-Messaging-Everyone/dp/1935182978)




