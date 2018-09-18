---

layout: post 
title: "python 和消息机制（一）：消息队列简介" 
excerpt: "简单介绍消息队列的基本概念和特性" 
categories: 程序技术
tags: [message_queue, amqp, pymq, python]
comments: true
share: true
---

简介
----

消息队列是生产者消费者模型的扩展，主要特点是：

1. 异步：把接受到的任务放到队列里面，以后处理
2. 分布式：消费者/worker 可以方便地横向扩展

关于消息队列，[这篇文章](http://blog.codepath.com/2013/01/06/asynchronous-processing-in-web-applications-part-2-developers-need-to-understand-message-queues/)有详细的介绍。

这个系列的文章主要介绍 python 消息队列的有关知识，会讲到下面几个概念：

1.	amqp: Adcanced Message Queue Protocol，[官网在这](https://www.amqp.org/)。
2.	rabbitmq: Erlang 语言的amqp 协议实现
3.	pyamqp：python message queue 的客户端，[github 页面](https://github.com/celery/py-amqp)
4.	kombu： message queue 的封装和抽象框架，实现了更好的接口、解决了不同客户端的一致性问题
5.	celery：使用 message queue 实现的分布式任务处理系统

**NOTE**：关于更多 message queue 的介绍和选择，可以参考 http://queues.io/ 网站，这里总结了所有常见的 message queue。

**NOTE**：关于其他的 python task queue，可以参考 [full stack python 的总结](http://www.fullstackpython.com/task-queues.html).

名词解释
--------

![](http://blog.codepath.com/wp-content/uploads/2012/11/mq_illustration_1.png) 最简单的消息模型可以抽象成上图中三个概念：

-	producer： 生产者负责产生和发送消息到 broker
-	broker：简单起见，你可以把 broker 理解为消息的处理中心
-	consumer： 消费者从 broker 中获取消息来处理。

上图的模型中，broker 只具备 message 存储功能，可能也包括重试、消息确认等特性，实际上的消息队列一般都有多个 queue，根据消息的不同特性把他们放到不同的地方，消费者也可以只处理自己感兴趣的消息，如下图所示。

![message queue](http://blog.codepath.com/wp-content/uploads/2012/11/mq_illustration_21.png)

实际中的例子可能是：某个网站在处理用户上传视频、发送通知、图像的压缩都放到不同的 queue里，由单独的消费者去处理；当然也有些情况下一个消息要交给多个消费者处理：比如用户上传视频之后，压缩视频的消费者和发送邮件通知的消费者都需要去处理。

-	queue：存放消息的队列，一般会按照消息的特点进行分类
-	fanout：把消息转发给多个消费者
-	direct：将消息转发给特定的消费者

更多特性
--------

处理前面已经提到的异步和可扩展两个主要的特性之外，消息队列还有其他的优点：

### 可靠性

消息队列一般会把接收到的消息存储到本地硬盘上，当消息被处理完之后从可能将其删除，这样即使应用挂掉或者消息队列本身挂掉，消息也能够重新加载。

### 松耦合

消息队列可以实现系统直接的解耦，不同的服务和系统之间可以通过消息队列进行通信，而不用关心彼此的实现细节，只要定义好消息的格式就行。

当然，除了上面的有点之外，message queue 也有缺点。消息队列是很重的组件，安装配置和维护会带来额外的工作量，对于小的系统来说，一般都没有引入消息队列的必要。

参考文档
--------

-	[Developers Need to Understand Message Queues](http://blog.codepath.com/2013/01/06/asynchronous-processing-in-web-applications-part-2-developers-need-to-understand-message-queues/)
