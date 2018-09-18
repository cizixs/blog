---
layout: post
title: "etcd 使用入门"
excerpt: "etcd 是 CoreOS 开发的配置管理和服务发现的项目，采用 raft 协议来实现分布式系统的一致性"
categories: blog
tags: [etcd, service discovery, http, raft]
comments: true
share: true
---

## 1. etcd 简介

coreos 开发的分布式服务系统，内部采用 raft 协议作为一致性算法。作为服务发现系统，有以下的特点：

- 简单：安装配置简单，而且提供了 HTTP API 进行交互，使用也很简单
- 安全：支持 SSL 证书验证
- 快速：根据[官方提供的 benchmark 数据](https://coreos.com/etcd/docs/latest/benchmarks/etcd-2-2-0-rc-benchmarks.html)，单实例支持每秒 2k+ 读操作
- 可靠：采用 [raft 算法](https://raft.github.io)，实现分布式系统数据的可用性和一致性

在这篇文章编写的时候，etcd 已经发布了 `3.0.4` 版本，被被用在 CoreOS、kubernetes、Cloud Foundry 等项目中。

etcd 目前默认使用 `2379` 端口提供 HTTP API 服务，`2380` 端口和 peer 通信（这两个端口已经被 [IANA 官方预留给 etcd](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=etcd)）；在之前的版本中，可能会分别使用 `4001` 和 `7001`，在使用的过程中需要注意这个区别。

虽然 etcd 也支持单点部署，但是在生产环境中推荐集群方式部署，一般 etcd 节点数会选择 3、5、7。etcd 会保证所有的节点都会保存数据，并保证数据的一致性和正确性。

## 2. 安装

因为 etcd 是 go 语言编写的，安装只需要下载对应的二进制文件，并放到合适的路径就行。

### 单点安装

如果在测试环境，启动一个单点的 etcd 服务，只需要运行 `etcd` 命令就行。

    ➜  etcd-v3.0.4-darwin-amd64 ./etcd
    2016-08-01 23:33:09.066930 I | etcdmain: etcd Version: 3.0.4
    2016-08-01 23:33:09.067085 I | etcdmain: Git SHA: d53923c
    2016-08-01 23:33:09.067092 I | etcdmain: Go Version: go1.6.3
    2016-08-01 23:33:09.067101 I | etcdmain: Go OS/Arch: darwin/amd64
    2016-08-01 23:33:09.067111 I | etcdmain: setting maximum number of CPUs to 4, total number of available CPUs is 4
    2016-08-01 23:33:09.067120 W | etcdmain: no data-dir provided, using default data-dir ./default.etcd
    2016-08-01 23:33:09.067803 I | etcdmain: listening for peers on http://localhost:2380
    2016-08-01 23:33:09.068123 I | etcdmain: listening for client requests on localhost:2379
    2016-08-01 23:33:09.069442 I | etcdserver: name = default
    2016-08-01 23:33:09.069465 I | etcdserver: data dir = default.etcd
    2016-08-01 23:33:09.069474 I | etcdserver: member dir = default.etcd/member
    2016-08-01 23:33:09.069480 I | etcdserver: heartbeat = 100ms
    2016-08-01 23:33:09.069485 I | etcdserver: election = 1000ms
    2016-08-01 23:33:09.069491 I | etcdserver: snapshot count = 10000
    2016-08-01 23:33:09.069510 I | etcdserver: advertise client URLs = http://localhost:2379
    2016-08-01 23:33:09.069523 I | etcdserver: initial advertise peer URLs = http://localhost:2380
    2016-08-01 23:33:09.069536 I | etcdserver: initial cluster = default=http://localhost:2380
    2016-08-01 23:33:09.258167 I | etcdserver: starting member 8e9e05c52164694d in cluster cdf818194e3a8c32
    2016-08-01 23:33:09.258927 I | raft: 8e9e05c52164694d became follower at term 0
    2016-08-01 23:33:09.259395 I | raft: newRaft 8e9e05c52164694d [peers: [], term: 0, commit: 0, applied: 0, lastindex: 0, lastterm: 0]
    2016-08-01 23:33:09.259594 I | raft: 8e9e05c52164694d became follower at term 1
    2016-08-01 23:33:09.331125 I | etcdserver: starting server... [version: 3.0.4, cluster version: to_be_decided]
    2016-08-01 23:33:09.331800 E | etcdserver: cannot monitor file descriptor usage (cannot get FDUsage on darwin)
    2016-08-01 23:33:09.334611 I | membership: added member 8e9e05c52164694d [http://localhost:2380] to cluster cdf818194e3a8c32
    2016-08-01 23:33:09.499412 I | raft: 8e9e05c52164694d is starting a new election at term 1
    2016-08-01 23:33:09.499450 I | raft: 8e9e05c52164694d became candidate at term 2
    2016-08-01 23:33:09.499462 I | raft: 8e9e05c52164694d received vote from 8e9e05c52164694d at term 2
    2016-08-01 23:33:09.499480 I | raft: 8e9e05c52164694d became leader at term 2
    2016-08-01 23:33:09.499493 I | raft: raft.node: 8e9e05c52164694d elected leader 8e9e05c52164694d at term 2
    2016-08-01 23:33:09.499702 I | etcdserver: setting up the initial cluster version to 3.0
    2016-08-01 23:33:09.507070 N | membership: set the initial cluster version to 3.0
    2016-08-01 23:33:09.507129 I | api: enabled capabilities for version 3.0
    2016-08-01 23:33:09.507211 I | etcdserver: published {Name:default ClientURLs:[http://localhost:2379]} to cluster cdf818194e3a8c32
    2016-08-01 23:33:09.507221 I | etcdmain: ready to serve client requests
    2016-08-01 23:33:09.507811 N | etcdmain: serving insecure client requests on localhost:2379, this is strongly discouraged!

从上面的输出中，我们可以看到很多信息：

- etcd 默认将数据存放到当前路径的  `default.etcd/` 目录下
- 在 `http://localhost:2380` 和集群中其他节点通信
- 在 `http://localhost:2379` 提供 HTTP API 服务，供客户端交互
- 该节点的名称默认为 `default`
- heartbeat 为 100ms，后面会说明这个配置的作用
- election 为 1000ms，后面会说明这个配置的作用
- snapshot count 为 10000，后面会说明这个配置的作用
- 集群和每个节点都会生成一个 uuid
- 启动的时候，会运行 raft，选举出 leader

### 集群安装

在安装和启动 etcd 服务的时候，各个节点需要知道集群中其他节点的信息（一般是 ip 和 port 信息）。根据你是否可以提前知道每个节点的 ip，有几种不同的启动方案：

- 静态配置：在启动 etcd server 的时候，通过 `--initial-cluster` 参数配置好所有的节点信息
- 使用已有的 etcd cluster 来注册和启动，比如官方提供的 `discovery.etcd.io`
- 使用 DNS 启动，

[etcd 的安装文档](https://coreos.com/etcd/docs/latest/clustering.html)官网已经给出了，这里不再赘述。当然，你也可以通过 docker 来安装 etcd，具体的文档可以看[这里](https://coreos.com/etcd/docs/latest/docker_guide.html)。

下面给出可以常用配置的参数和它们的解释，方便理解：

- `--name`：方便理解的节点名称，默认为 `default`，在集群中应该保持唯一，可以使用 hostname
- `--data-dir`：服务运行数据保存的路径，默认为 `${name}.etcd`
- `--snapshot-count`：指定有多少事务（transaction）被提交时，触发截取快照保存到磁盘
- `--heartbeat-interval`：leader 多久发送一次心跳到 followers。默认值是 100ms
- `--eletion-timeout`：重新投票的超时时间，如果 follow 在该时间间隔没有收到心跳包，会触发重新投票，默认为 1000 ms
- `--listen-peer-urls`：和同伴通信的地址，比如 `http://ip:2380`，如果有多个，使用逗号分隔。需要所有节点都能够访问，**所以不要使用 localhost！**
- `--listen-client-urls`：对外提供服务的地址：比如 `http://ip:2379,http://127.0.0.1:2379`，客户端会连接到这里和 etcd 交互
- `--advertise-client-urls`：对外公告的该节点客户端监听地址，这个值会告诉集群中其他节点
- `--initial-advertise-peer-urls`：该节点同伴监听地址，这个值会告诉集群中其他节点
- `--initial-cluster`：集群中所有节点的信息，格式为 `node1=http://ip1:2380,node2=http://ip2:2380,…`。注意：这里的 `node1` 是节点的 `--name` 指定的名字；后面的 `ip1:2380` 是 `--initial-advertise-peer-urls` 指定的值
- `--initial-cluster-state`：新建集群的时候，这个值为 `new`；假如已经存在的集群，这个值为 `existing`
- `--initial-cluster-token`：创建集群的 token，这个值每个集群保持唯一。这样的话，如果你要重新创建集群，即使配置和之前一样，也会再次生成新的集群和节点 uuid；否则会导致多个集群之间的冲突，造成未知的错误

所有以 `--init` 开头的配置都是在 bootstrap 集群的时候才会用到，后续节点的重启会被忽略。

NOTE：所有的参数也可以通过环境变量进行设置，`--my-flag` 对应环境变量的 `ETCD_MY_FLAG`；但是命令行指定的参数会覆盖环境变量对应的值。

在后面的文章中，为了简单起见，我们采用了单点的 etcd server，**请在生产环境中配置 etcd 集群，并使用 SSL 安全机制**。

## 3. etcd 基础知识

每个 etcd cluster 都是有若干个 member 组成的，每个 member 是一个独立运行的 etcd 实例，单台机器上可以运行多个 member。

在正常运行的状态下，集群中会有一个 leader，其余的 member 都是 followers。leader 向 followers 同步日志，保证数据在各个 member 都有副本。leader 还会定时向所有的 member 发送心跳报文，如果在规定的时间里 follower 没有收到心跳，就会重新进行选举。

客户端所有的请求都会先发送给 leader，leader 向所有的 followers 同步日志，等收到超过半数的确认后就把该日志存储到磁盘，并返回响应客户端。

每个 etcd 服务有三大主要部分组成：raft 实现、WAL 日志存储、数据的存储和索引。WAL 会在本地磁盘（就是之前提到的 `--data-dir`）上存储日志内容（wal file）和快照（snapshot）。

## 4. API 文档

etcd 对外通过 HTTP API 对外提供服务，这种方式方便测试（通过 curl 或者其他工具就能和 etcd 交互），也很容易集成到各种语言中（每个语言封装 HTTP API 实现自己的 client 就行）。

这个部分，我们就介绍 etcd 通过 HTTP API 提供了哪些功能，并使用 [`httpie` ](https://github.com/jkbrzt/httpie)来交互（当然你也可以使用 curl 或者其他工具）。

### 获取 etcd 服务的版本信息

    ➜  http http://127.0.0.1:2379/version
    HTTP/1.1 200 OK
    Content-Length: 44
    Content-Type: application/json
    Date: Tue, 02 Aug 2016 04:27:32 GMT

    {
        "etcdcluster": "3.0.0",
        "etcdserver": "3.0.4"
    }

### key 的增删查改

etcd 的数据按照树形的结构组织，类似于 linux 的文件系统，也有目录和文件的区别，不过一般被称为 nodes。数据的 endpoint 都是以 `/v2/keys` 开头（v2 表示当前 API 的版本），比如 `/v2/keys/names/cizixs`。

要创建一个值，只要使用 `PUT` 方法在对应的 url endpoint 设置就行。如果对应的 key 已经存在， `PUT` 也会对 key 进行更新。

    ➜  http PUT http://127.0.0.1:2379/v2/keys/message value=="hello, etcd"
    HTTP/1.1 201 Created
    Content-Length: 100
    Content-Type: application/json
    Date: Tue, 02 Aug 2016 04:48:04 GMT
    X-Etcd-Cluster-Id: cdf818194e3a8c32
    X-Etcd-Index: 4
    X-Raft-Index: 28429
    X-Raft-Term: 2

    {
        "action": "set",
        "node": {
            "createdIndex": 4,
            "key": "/message",
            "modifiedIndex": 4,
            "value": "hello, etcd"
        }
    }

上面这个命令通过 `PUT` 方法把 `/message` 设置为 `hello, etcd`。返回的格式中，各个字段的意义是：

- `action`：请求出发的动作，这里因为是新建一个 key 并设置它的值，所以是 `set`
- `node.key`：key 的 HTTP 路径
- `node.value`：请求处理之后，key 的值
- `node.createdIndex`： createdIndex 是一个递增的值，每次有 key 被创建的时候会增加
- `node.modifiedIndex`：同上，只不过每次有 key 被修改的时候增加

除返回的 json 体外，上面的情况还包含了一些特殊的  HTTP 头部信息，这些信息说明了 etcd cluster 的一些情况。它们的具体含义如下：

- `X-Etcd-Index`：当前 etcd 集群的 index
- `X-Raft-Index`：raft 集群的 index
- `X-Raft-Term`：raft 集群的任期，每次有 leader 选举的时候，这个值就会增加

查看信息比较简单，使用 `GET` 方法，url 指向要查看的值就行：

    ➜  http GET http://127.0.0.1:2379/v2/keys/message
    HTTP/1.1 200 OK
    Content-Length: 97
    Content-Type: application/json
    Date: Tue, 02 Aug 2016 05:23:14 GMT
    X-Etcd-Cluster-Id: cdf818194e3a8c32
    X-Etcd-Index: 7
    X-Raft-Index: 30801
    X-Raft-Term: 2

    {
        "action": "get",
        "node": {
            "createdIndex": 7,
            "key": "/message",
            "modifiedIndex": 7,
            "value": "hello, etcd"
        }
    }

这里的 `action` 变成了 `get`，其他返回的值和上面的含义一样，略过不提。

NOTE：这两个命令并不是连着执行的，中间我有执行其他操作，因此 `index` 会出现不连续的情况。

前面已经提过， `PUT` 也可用来更新 key 的值我们就来看看例子。

    ➜  http PUT http://127.0.0.1:2379/v2/keys/message value=="I'm changed"
    HTTP/1.1 200 OK
    Content-Length: 184
    Content-Type: application/json
    Date: Tue, 02 Aug 2016 05:28:17 GMT
    X-Etcd-Cluster-Id: cdf818194e3a8c32
    X-Etcd-Index: 8
    X-Raft-Index: 31407
    X-Raft-Term: 2

    {
        "action": "set",
        "node": {
            "createdIndex": 8,
            "key": "/message",
            "modifiedIndex": 8,
            "value": "I'm changed"
        },
        "prevNode": {
            "createdIndex": 7,
            "key": "/message",
            "modifiedIndex": 7,
            "value": "hello, etcd"
        }
    }

和第一次执行 `PUT` 命令不同的是，返回中多了一个字段 `prevNode`，它保存着更新之前该 key 的信息。它的格式和 `node` 是一样的，如果之前没有这个信息，这个字段会被省略。

删除 key 可以通过 `DELETE` 方法，，比如我们要删除上面创建的字段：

    ➜  http DELETE http://127.0.0.1:2379/v2/keys/message
    HTTP/1.1 200 OK
    Content-Length: 168
    Content-Type: application/json
    Date: Tue, 02 Aug 2016 05:31:56 GMT
    X-Etcd-Cluster-Id: cdf818194e3a8c32
    X-Etcd-Index: 9
    X-Raft-Index: 31847
    X-Raft-Term: 2

    {
        "action": "delete",
        "node": {
            "createdIndex": 8,
            "key": "/message",
            "modifiedIndex": 9
        },
        "prevNode": {
            "createdIndex": 8,
            "key": "/message",
            "modifiedIndex": 8,
            "value": "I'm changed"
        }
    }

注意，这里的 `action` 是 `delete`，并且 `modifiedIndex` 增加了，但是 `createdIndex` 没有变化，因为这是一个修改操作，不是新建操作。

### TTL

etcd 中，key 可以有 TTL 属性，超过这个时间会被自动删除。我们来设置一个看看：

    ➜  http PUT http://127.0.0.1:2379/v2/keys/tempkey value=="Gone with wind" ttl==5

    HTTP/1.1 201 Created
    Content-Length: 159
    Content-Type: application/json
    Date: Tue, 02 Aug 2016 05:48:17 GMT
    X-Etcd-Cluster-Id: cdf818194e3a8c32
    X-Etcd-Index: 10
    X-Raft-Index: 33810
    X-Raft-Term: 2

    {
        "action": "set",
        "node": {
            "createdIndex": 10,
            "expiration": "2016-08-02T05:48:22.618695843Z",
            "key": "/tempkey",
            "modifiedIndex": 10,
            "ttl": 5,
            "value": "Gone with wind"
        }
    }

除了一般 key 返回的信息之外，上面多了两个字段：

- `expiration`：代表 key 过期被删除的时间
- `ttl`：表示 key 还要多少秒可以存活（这个值是动态的，会根据你请求的时候和过期时间进行计算）

如果我们在 5s 之后再去请求查看该 key，会发现报错信息：

    ➜  http http://127.0.0.1:2379/v2/keys/tempkey
    HTTP/1.1 404 Not Found
    Content-Length: 74
    Content-Type: application/json
    Date: Tue, 02 Aug 2016 05:48:28 GMT
    X-Etcd-Cluster-Id: cdf818194e3a8c32
    X-Etcd-Index: 11

    {
        "cause": "/tempkey",
        "errorCode": 100,
        "index": 11,
        "message": "Key not found"
    }

http 返回为 `404`，并且返回体中给出了 `errorCode` 和错误信息。

TTL 也可通过 `PUT` 方法进行取消，只要设置空值 `ttl=` 就行，这样 key 就不会过期被删除。比如：

    ➜  http PUT http://127.0.0.1:2379/v2/keys/foo value==bar ttl== prevExist==true

注意：**需要设置 value==bar，不然 key 会变成空值。**

如果只是想更新 TTL，可以添加上  `refresh==true` 参数：

    ➜  etcd-v3.0.4-darwin-amd64 http -v PUT http://127.0.0.1:2379/v2/keys/tempkey

    HTTP/1.1 200 OK
    Content-Length: 305
    Content-Type: application/json
    Date: Tue, 02 Aug 2016 06:05:12 GMT
    X-Etcd-Cluster-Id: cdf818194e3a8c32
    X-Etcd-Index: 20
    X-Raft-Index: 35849
    X-Raft-Term: 2

    {
        "action": "set",
        "node": {
            "createdIndex": 20,
            "expiration": "2016-08-02T06:13:32.370495212Z",
            "key": "/tempkey",
            "modifiedIndex": 20,
            "ttl": 500,
            "value": "hello, there"
        },
        "prevNode": {
            "createdIndex": 19,
            "expiration": "2016-08-02T06:10:05.366042396Z",
            "key": "/tempkey",
            "modifiedIndex": 19,
            "ttl": 293,
            "value": "hello, there"
        }
    }

### 监听变化

etcd 提供了监听的机制，可以让客户端使用 long pulling 监听某个 key，当发生变化的时候接接收通知因为 etcd 经常被用作服务发现，集群中的信息有更新的时候需要及时被检测，做出对应的处理。因此需要有监听机制，来告诉客户端特定 key 的变化情况。

监听动作只需要 `GET` 方法，添加上 `wait=true` 参数就行.使用 `recursive=true` 参数，也能监听某个目录。

    ➜  http http://127.0.0.1:2379/v2/keys/foo wait==true
    HTTP/1.1 200 OK
    Content-Type: application/json
    Date: Tue, 02 Aug 2016 06:09:47 GMT
    Transfer-Encoding: chunked
    X-Etcd-Cluster-Id: cdf818194e3a8c32
    X-Etcd-Index: 22
    X-Raft-Index: 36401
    X-Raft-Term: 2


这个时候，客户端会阻塞在这里，如果在另外的 terminal 修改 key 的值，监听的客户端会接收到消息，打印出更新的值：

    {
        "action": "set",
        "node": {
            "createdIndex": 23,
            "key": "/foo",
            "modifiedIndex": 23,
            "value": "changed"
        },
        "prevNode": {
            "createdIndex": 22,
            "key": "/foo",
            "modifiedIndex": 22,
            "value": "bar"
        }
    }

除了这种最简单的监听之外，还可以提供基于 index 的监听。如果通过 `waitIndex` 指定了 index，那么

 会返回从 index 开始出现的第一个事件，这包含了两种情况：

- 给出的 index 小于等于当前 index ，即事件已经发生，那么监听会立即返回该事件
- 给出的 index 大于当前 index，等待 index 之后的事件发生并返回

目前 etcd 只会保存最近 1000 个事件（整个集群范围内），再早之前的事件会被清理，如果监听被清理的事件会报错。如果出现漏过太多事件（超过 1000）的情况，需要重新获取当然的 index 值（`X-Etcd-Index`），然后从 `X-Etcd-Index+1` 开始监听。

因为监听的时候出现事件就会直接返回，因此需要客户端编写循环逻辑保持监听状态。在两次监听的间隔中出现的事件，很可能被漏过。所以最好把事件处逻辑做成异步的，不要阻塞监听逻辑。

注意：**监听 key 时会出现因为长时间没有返回导致连接被 close 的情况，客户端需要处理这种错误并自动重试。**

### 自动创建有序的 keys

在有些情况下，我们需要 key 是有序的，etcd 提供了这个功能。对某个目录使用 `POST` 方法，能自动生成有序的 key，这种模式可以用于队列处理等场景。

    ➜  http POST http://127.0.0.1:2379/v2/keys/queue value==job1
    HTTP/1.1 201 Created
    Content-Length: 121
    Content-Type: application/json
    Date: Tue, 02 Aug 2016 07:08:38 GMT
    X-Etcd-Cluster-Id: cdf818194e3a8c32
    X-Etcd-Index: 1030
    X-Raft-Index: 44470
    X-Raft-Term: 2

    {
        "action": "create",
        "node": {
            "createdIndex": 1030,
            "key": "/queue/00000000000000001030",
            "modifiedIndex": 1030,
            "value": "job1"
        }
    }

创建的 key 会使用 etcd index，只能保证递增，无法保证是连续的（因为两次创建之间可能会有其他发生）。然后用相同的命令创建多个值，在获取值的时候使用 `sorted=true`参数就会返回已经排序的值：

    ➜  http http://127.0.0.1:2379/v2/keys/queue sorted==true
    HTTP/1.1 200 OK
    Content-Length: 385
    Content-Type: application/json
    Date: Tue, 02 Aug 2016 07:11:32 GMT
    X-Etcd-Cluster-Id: cdf818194e3a8c32
    X-Etcd-Index: 1032
    X-Raft-Index: 44819
    X-Raft-Term: 2

    {
        "action": "get",
        "node": {
            "createdIndex": 1030,
            "dir": true,
            "key": "/queue",
            "modifiedIndex": 1030,
            "nodes": [
                {
                    "createdIndex": 1030,
                    "key": "/queue/00000000000000001030",
                    "modifiedIndex": 1030,
                    "value": "job1"
                },
                {
                    "createdIndex": 1031,
                    "key": "/queue/00000000000000001031",
                    "modifiedIndex": 1031,
                    "value": "job2"
                },
                {
                    "createdIndex": 1032,
                    "key": "/queue/00000000000000001032",
                    "modifiedIndex": 1032,
                    "value": "job3"
                }
            ]
        }
    }

### 设置目录的 TTL

和 key 类似，目录（dir）也可以有过期时间。设置的方法也一样，只不过多了 `dir=true` 参数来说明这是一个目录。

    ➜  http PUT http://127.0.0.1:2379/v2/keys/dir dir==true ttl==5 prevExist==true
    HTTP/1.1 200 OK
    Content-Length: 226
    Content-Type: application/json
    Date: Tue, 02 Aug 2016 07:15:42 GMT
    X-Etcd-Cluster-Id: cdf818194e3a8c32
    X-Etcd-Index: 1033
    X-Raft-Index: 45325
    X-Raft-Term: 2

    {
        "action": "update",
        "node": {
            "createdIndex": 1029,
            "dir": true,
            "expiration": "2016-08-02T07:15:47.970434032Z",
            "key": "/dir",
            "modifiedIndex": 1033,
            "ttl": 5
        },
        "prevNode": {
            "createdIndex": 1029,
            "dir": true,
            "key": "/dir",
            "modifiedIndex": 1029
        }
    }

目录过期的时候会被自动删除，包括它里面所有的子目录和 key，所有监听这个目录中内容的客户端都会收到对应的事件。

### 比较更新的原子操作

在分布式环境中，我们需要解决多个客户端的竞争问题，etcd 提供了原子操作 `CompareAndSwap`（CAS），通过这个操作可以很容易实现分布式锁。

简单来说，这个命令只有在客户端提供的条件成立的情况下才会更新对应的值。目前支持的条件包括：

- `preValue`：检查 key 之前的值是否和客户端提供的一致
- `prevIndex`：检查 key 之前的 `modifiedIndex` 是否和客户端提供的一致
- `prevExist`：检查 key 是否已经存在。如果存在就执行更新操作，如果不存在，执行 create 操作

举个栗子，比如目前 `/foo` 的值为 `bar`，要把它更新成 `changed`，可以使用：

    ➜  http PUT http://127.0.0.1:2379/v2/keys/foo prevValue==bar value==changed
    HTTP/1.1 200 OK
    Content-Length: 190
    Content-Type: application/json
    Date: Tue, 02 Aug 2016 07:37:05 GMT
    X-Etcd-Cluster-Id: cdf818194e3a8c32
    X-Etcd-Index: 1036
    X-Raft-Index: 47893
    X-Raft-Term: 2

    {
        "action": "compareAndSwap",
        "node": {
            "createdIndex": 1035,
            "key": "/foo",
            "modifiedIndex": 1036,
            "value": "changed"
        },
        "prevNode": {
            "createdIndex": 1035,
            "key": "/foo",
            "modifiedIndex": 1035,
            "value": "bar"
        }
    }

如果提供的条件不对，会报 `412` 错误：

    ➜ http PUT http://127.0.0.1:2379/v2/keys/foo prevValue==bar value==new
    HTTP/1.1 412 Precondition Failed
    Content-Length: 85
    Content-Type: application/json
    Date: Tue, 02 Aug 2016 07:37:38 GMT
    X-Etcd-Cluster-Id: cdf818194e3a8c32
    X-Etcd-Index: 1036

    {
        "cause": "[bar != changed]",
        "errorCode": 101,
        "index": 1036,
        "message": "Compare failed"
    }

注意：匹配条件是 `prevIndex=0` 的话，也会通过检查。

这些条件也可以组合起来使用，只有当都满足的时候，才会执行对应的操作。

### 比较删除的原子操作

和条件更新类似，etcd 也支持条件删除操作：只有在客户端提供的条件成立的情况下，才会执行删除操作。支持 `prevValue` 和 `prevIndex` 两种条件检查，没有 `prevExist`，因为删除不存在的值本身就会报错。

我们来删除上面例子中更新的 `/foo` ，先看一下提供的条件不对的情况：

    ➜  http DELETE http://127.0.0.1:2379/v2/keys/foo prevValue==bar
    HTTP/1.1 412 Precondition Failed
    Content-Length: 85
    Content-Type: application/json
    Date: Tue, 02 Aug 2016 07:49:13 GMT
    X-Etcd-Cluster-Id: cdf818194e3a8c32
    X-Etcd-Index: 1043

    {
        "cause": "[bar != changed]",
        "errorCode": 101,
        "index": 1043,
        "message": "Compare failed"
    }

如果提供的条件成立，对应的 key 就会被删除：

    ➜  http DELETE http://127.0.0.1:2379/v2/keys/foo prevValue==changed
    HTTP/1.1 200 OK
    Content-Length: 178
    Content-Type: application/json
    Date: Tue, 02 Aug 2016 07:51:27 GMT
    X-Etcd-Cluster-Id: cdf818194e3a8c32
    X-Etcd-Index: 1044
    X-Raft-Index: 49629
    X-Raft-Term: 2

    {
        "action": "compareAndDelete",
        "node": {
            "createdIndex": 1043,
            "key": "/foo",
            "modifiedIndex": 1044
        },
        "prevNode": {
            "createdIndex": 1043,
            "key": "/foo",
            "modifiedIndex": 1043,
            "value": "changed"
        }
    }

### 操作目录

在创建 key 的时候，如果它所在路径的目录不存在，会自动被创建，所以在多数情况下我们不需要关心目录的创建。目录的操作和 key 的操作基本一致，唯一的区别是需要加上 `dir=true` 参数指明操作的对象是目录。

比如，如果想要显示地创建目录，可以使用 `PUT` 方法，并设置 `dir=true`：

    ➜ http PUT http://127.0.0.1:2379/v2/keys/anotherdir dir==true
    HTTP/1.1 201 Created
    Content-Length: 98
    Content-Type: application/json
    Date: Tue, 02 Aug 2016 07:53:48 GMT
    X-Etcd-Cluster-Id: cdf818194e3a8c32
    X-Etcd-Index: 1045
    X-Raft-Index: 49914
    X-Raft-Term: 2

    {
        "action": "set",
        "node": {
            "createdIndex": 1045,
            "dir": true,
            "key": "/anotherdir",
            "modifiedIndex": 1045
        }
    }

 创建目录的操作不能重复执行，再次执行上面的命令会报  `HTTP 403` 错误。

如果 `GET` 方法对应的 url 是目录的话，etcd 会列出该目录所有节点的信息（不需要指定 `dir=true`）。比如要列出根目录下所有的节点：

    ➜  http http://127.0.0.1:2379/v2/keys/
    HTTP/1.1 200 OK
    Content-Length: 190
    Content-Type: application/json
    Date: Tue, 02 Aug 2016 07:55:41 GMT
    X-Etcd-Cluster-Id: cdf818194e3a8c32
    X-Etcd-Index: 1045
    X-Raft-Index: 50141
    X-Raft-Term: 2

    {
        "action": "get",
        "node": {
            "dir": true,
            "nodes": [
                {
                    "createdIndex": 1045,
                    "dir": true,
                    "key": "/anotherdir",
                    "modifiedIndex": 1045
                },
                {
                    "createdIndex": 1030,
                    "dir": true,
                    "key": "/queue",
                    "modifiedIndex": 1030
                }
            ]
        }
    }

如果添加上 `recursive=true` 参数，就会递归地列出所有的值：

    ➜  http http://127.0.0.1:2379/v2/keys/\?recursive\=true
    HTTP/1.1 200 OK
    Content-Length: 482
    Content-Type: application/json
    Date: Tue, 02 Aug 2016 07:57:48 GMT
    X-Etcd-Cluster-Id: cdf818194e3a8c32
    X-Etcd-Index: 1045
    X-Raft-Index: 50394
    X-Raft-Term: 2

    {
        "action": "get",
        "node": {
            "dir": true,
            "nodes": [
                {
                    "createdIndex": 1045,
                    "dir": true,
                    "key": "/anotherdir",
                    "modifiedIndex": 1045
                },
                {
                    "createdIndex": 1030,
                    "dir": true,
                    "key": "/queue",
                    "modifiedIndex": 1030,
                    "nodes": [
                        {
                            "createdIndex": 1031,
                            "key": "/queue/00000000000000001031",
                            "modifiedIndex": 1031,
                            "value": "job2"
                        },
                        {
                            "createdIndex": 1032,
                            "key": "/queue/00000000000000001032",
                            "modifiedIndex": 1032,
                            "value": "job3"
                        },
                        {
                            "createdIndex": 1030,
                            "key": "/queue/00000000000000001030",
                            "modifiedIndex": 1030,
                            "value": "job1"
                        }
                    ]
                }
            ]
        }
    }

和 linux 删除目录的设计一样，要区别空目录和非空目录。删除空目录很简单，使用 `DELETE` 方法，并添加上 `dir=true` 参数，类似于 `rmdir`；而对于非空目录，需要添加上 `recursive=true`，类似于 `rm -rf`。

    ➜  http DELETE http://127.0.0.1:2379/v2/keys/queue dir==true
    HTTP/1.1 403 Forbidden
    Content-Length: 80
    Content-Type: application/json
    Date: Tue, 02 Aug 2016 08:06:44 GMT
    X-Etcd-Cluster-Id: cdf818194e3a8c32
    X-Etcd-Index: 1045

    {
        "cause": "/queue",
        "errorCode": 108,
        "index": 1045,
        "message": "Directory not empty"
    }

    ➜  http DELETE http://127.0.0.1:2379/v2/keys/queue dir==true recursive==true
    HTTP/1.1 200 OK
    Content-Length: 176
    Content-Type: application/json
    Date: Tue, 02 Aug 2016 08:06:48 GMT
    X-Etcd-Cluster-Id: cdf818194e3a8c32
    X-Etcd-Index: 1046
    X-Raft-Index: 51478
    X-Raft-Term: 2

    {
        "action": "delete",
        "node": {
            "createdIndex": 1030,
            "dir": true,
            "key": "/queue",
            "modifiedIndex": 1046
        },
        "prevNode": {
            "createdIndex": 1030,
            "dir": true,
            "key": "/queue",
            "modifiedIndex": 1030
        }
    }

### 隐藏的节点

etcd 中节点也可以是默认隐藏的，类似于 linux 中以 `.` 开头的文件或者文件夹，以 `_` 开头的节点也是默认隐藏的，不会在列出目录的时候显示。只有知道隐藏节点的完整路径，才能够访问它的信息。

### 查看集群数据信息

etcd 还保存了集群的数据信息，包括节点之间的网络信息，操作的统计信息。

- `/v2/stats/leader`会返回集群中 leader 的信息，以及 followers 的基本信息
- `/v2/stats/self` 会返回当前节点的信息
- `/v2/state/store`：会返回各种命令的统计信息

### 成员管理

etcd 在 `/v2/members` 下保存着集群中各个成员的信息，

    ➜  http http://127.0.0.1:2379/v2/members
    HTTP/1.1 200 OK
    Content-Length: 133
    Content-Type: application/json
    Date: Tue, 02 Aug 2016 08:15:56 GMT
    X-Etcd-Cluster-Id: cdf818194e3a8c32

    {
        "members": [
            {
                "clientURLs": [
                    "http://localhost:2379"
                ],
                "id": "8e9e05c52164694d",
                "name": "default",
                "peerURLs": [
                    "http://localhost:2380"
                ]
            }
        ]
    }

可以通过 `POST` 方法添加成员：

    curl http://10.0.0.10:2379/v2/members -XPOST \
    -H "Content-Type: application/json" -d '{"peerURLs":["http://10.0.0.10:2380"]}'

 也可以通过 `DELETE` 方法删除成员：

    curl http://10.0.0.10:2379/v2/members/272e204152 -XDELETE

或者通过 PUT 更新成员的 peer url：

    curl http://10.0.0.10:2379/v2/members/272e204152 -XPUT \
    -H "Content-Type: application/json" -d '{"peerURLs":["http://10.0.0.10:2380"]}'

## 5. etcdctl 命令行工具

除了 HTTP API 外，etcd 还提供了 `etcdctl` 命令行工具和 etcd 服务交互。这个命令行用 go 语言编写，也是对 HTTP API 的封装，日常使用起来也更容易。

etcdctl 的安装就不说了，从官网下载二进制文件放到系统的 PATH 路径下就行了。

    # 设置一个 key 的值
    ➜ ./etcdctl set /message "hello, etcd"
    hello, etcd

    # 获取 key 的值
    ➜ ./etcdctl get /message
    hello, etcd

    # 获取 key 的值，包含更详细的元数据
    ➜  ./etcdctl -o extended get /message
    Key: /message
    Created-Index: 1073
    Modified-Index: 1073
    TTL: 0
    Index: 1073

    hello, etcd

    # 获取不存在 key 的值，会报错
    ➜  ./etcdctl get /notexist
    Error:  100: Key not found (/notexist) [1048]

    # 设置 key 的 ttl，过期后会被自动删除
    ➜  ./etcdctl set /tempkey "gone with wind" --ttl 5
    gone with wind
    ➜  ./etcdctl get /tempkey
    gone with wind
    ➜  ./etcdctl get /tempkey
    Error:  100: Key not found (/tempkey) [1050]

    # 如果 key 的值是 "hello, etcd"，就把它替换为 "goodbye, etcd"
    ➜  ./etcdctl set --swap-with-value "hello, world" /message "goodbye, etcd"
    Error:  101: Compare failed ([hello, world != hello, etcd]) [1050]
    ➜  ./etcdctl set --swap-with-value "hello, etcd" /message "goodbye, etcd"
    goodbye, etcd

    # 仅当 key 不存在的时候创建
    ➜  ./etcdctl mk /foo bar
    bar
    ➜  ./etcdctl mk /foo bar
    Error:  105: Key already exists (/foo) [1052]

    # 自动创建排序的 key
    ➜  ./etcdctl mk --in-order /queue job1
    job1
    ➜  ./etcdctl mk --in-order /queue job2
    job2
    ➜  ./etcdctl ls --sort /queue
    /queue/00000000000000001053
    /queue/00000000000000001054

    # 更新 key 的值或者 ttl，只有当 key 已经存在的时候才会生效，否则报错
    ➜  ./etcdctl update /message "I'am changed"
    I'am changed
    ➜  ./etcdctl get /message
    I'am changed
    ➜  ./etcdctl update /notexist "I'am changed"
    Error:  100: Key not found (/notexist) [1055]
    ➜  ./etcdctl update --ttl 3 /message "I'am changed"
    I'am changed
    ➜  ./etcdctl get /message
    Error:  100: Key not found (/message) [1057]

    # 删除某个 key
    ➜  ./etcdctl mk /foo bar
    bar
    ➜  ./etcdctl rm /foo
    PrevNode.Value: bar
    ➜  ./etcdctl get /foo
    Error:  100: Key not found (/foo) [1062]

    # 只有当 key 的值匹配的时候，才进行删除
    ➜  ./etcdctl mk /foo bar
    bar
    ➜  ./etcdctl rm --with-value wrong /foo
    Error:  101: Compare failed ([wrong != bar]) [1063]
    ➜  ./etcdctl rm --with-value bar /foo

    # 创建一个目录
    ➜  ./etcdctl mkdir /dir

    # 删除空目录
    ➜  ./etcdctl mkdir /dir/subdir/
    ➜  ./etcdctl rmdir /dir/subdir/

    # 删除非空目录
    ➜  ./etcdctl rmdir /dir
    Error:  108: Directory not empty (/dir) [1071]
    ➜  ./etcdctl rm --recursive /dir

    # 列出目录的内容
    ➜  ./etcdctl ls /
    /queue
    /anotherdir
    /message

    # 递归列出目录的内容
    ➜  ./etcdctl ls --recursive /
    /anotherdir
    /message
    /queue
    /queue/00000000000000001053
    /queue/00000000000000001054

    # 监听某个 key，当 key 改变的时候会打印出变化
    ➜  ./etcdctl watch /message
    changed

    # 监听某个目录，当目录中任何 node 改变的时候，都会打印出来
    ➜  ./etcdctl watch --recursive /
    [set] /message
    changed

    # 一直监听，除非 `CTL + C` 导致退出监听
    ➜  ./etcdctl watch --forever /message
    new value
    chaned again
    Wola

    # 监听目录，并在发生变化的时候执行一个命令
    ➜  ./etcdctl exec-watch --recursive / -- sh -c "echo change detected."
    change detected.
    change detected.

## 6. 总结

etcd 默认只保存 1000 个历史事件，所以不适合有大量更新操作的场景，这样会导致数据的丢失。 etcd 典型的应用场景是配置管理和服务发现，这些场景都是读多写少的。

相比于 zookeeper，etcd 使用起来要简单很多。不过要实现真正的服务发现功能，etcd 还需要和其他工具（比如 registrator、confd 等）一起使用来实现服务的自动注册和更新。

目前 etcd 还没有图形化的工具。

## 7. 参考资料

- [CoreOS 实战：剖析 etcd](http://www.infoq.com/cn/articles/coreos-analyse-etcd)

- [CoreOS实践指南（五）：分布式数据存储Etcd（上）](http://www.csdn.net/article/2015-01-22/2823659)

- [CoreOS实践指南（六）：分布式数据存储Etcd（下）](http://www.csdn.net/article/2015-01-28/2823739)

- [etcd API document](https://coreos.com/etcd/docs/latest/api.html)

- [etcd members API document](https://coreos.com/etcd/docs/latest/members_api.html)

- [etcd configuration](https://github.com/coreos/etcd/blob/master/Documentation/op-guide/configuration.md)
