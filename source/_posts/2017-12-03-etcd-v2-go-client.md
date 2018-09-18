---
layout: post
title: "etcd go 语言 v2 客户端开发介绍"
excerpt: "很久之前写过一篇 etcd 的介绍文章，主要是讲 etcd 的概念和使用方式，这篇文章介绍如何使用 go 语言进行 etcd 的开发工作。"
categories: blog
tags: [etcd, go]
comments: true
share: true
---

## etcd 介绍

很久之前写过[一篇 etcd 的介绍文章](http://cizixs.com/2016/08/02/intro-to-etcd)，主要是讲 etcd 的概念和使用方式，这篇文章介绍如何使用 go 语言进行 etcd 的开发工作。

etcd 目前最新版 API 是 v3 版本，之前被广泛使用的 API 为 v2 版本。这篇文章会介绍 v2 版本 go 语言客户端的使用，涉及了最常见的增删查改和监听操作。

## v2 版本使用

[之前的文章](http://cizixs.com/2016/08/02/intro-to-etcd) 已经对 etcd v2 版本的概念和 API 介绍得很详细了，这里再总结一下。

- 作为分布式的键值存储，etcd v2 存储的 key 和 value 都是字符串类型。其中 key 以类 unix 文件系统的结构存储，分为目录和文件，目录下面可以嵌套目录和文件，文件只能对应某个值（value）。可以对 key 进行创建、删除和修改的工作
- 可以为 key（包括目录和文件）设置一个 TTL 值，也就是过期时间，等到时间过了，值自动被删除
- 提供了 CAS(compare And Set）和 CAD（Compare And Delete）两种原子操作，可以实现比较然后执行某个动作的逻辑
- 可以监听某个文件和目录，当里面的值发生变化时，监听者会立即收到通知

概念讲完了，下面看看如何用 go 语言进行 etcd 的开发工作。etcd 官方就维护了一个 etcd go 语言客户端，在 `github.com/coreos/etcd/client`，我们可以用下面的语句把它导入到 go 程序中：

```
import	(
    ...
    etcd "github.com/coreos/etcd/client"
    "github.com/coreos/etcd/pkg/transport"
)
```

我们给它取了个别名 `etcd`，之后后面的程序中我们可以用 `etcd` 而不是 `client` 来使用它提供的功能（这样程序的可读性更好）。


### 创建 client 客户端

首先，我们要创建一个 etcd 的 Client，可以使用 `etcd.New()` 函数，它接受 `etcd.Config` 作为参数，后者主要的参数包括：

- `Endpoints`：etcd server 集群的地址列表，是一个 string 列表，每个值为一个 etcd 的监听地址
- `Transport`：底层用于进行 HTTP 请求传输的结构体
- `Username` 和 `Password`：进行简单认证的用户名和密码

其他还有一些参数这里就不说了，使用默认值就行。为了完整性，我们会处理 etcd server 开启了简单认证和 TLS 的情况，首先创建一个自定义的结构体保存所有需要的参数：

```
type EtcdConfig struct {
	Endpoints []string
	KeyFile   string
	CertFile  string
	CAFile    string
	Username  string
	Password  string
}
```

注意这是我们自己定义的结构体，不是客户端程序提供的。

然后构建能进行 TLS 处理的 transport，`transport` 是 etcd 提供的库，它内部会处理证书的读取和验证工作：

```
tlsInfo := transport.TLSInfo{
	CertFile: c.CertFile,
	KeyFile:  c.KeyFile,
	CAFile:   c.CAFile,
}

t, err := transport.NewTransport(tlsInfo, time.Second)
```

然后就能构建 client：

```
client, err := etcd.New(etcd.Config{
	Endpoints: c.Endpoints,
	Transport: t,
	Username:  c.Username,
	Password:  c.Password,
})
```

这个 client 主要负责 HTTP API 交互的逻辑，同时也负责从多个 endpoints 中选择一个可用的来调用。但是和 keys 交互的逻辑并没有直接在这个 client 中，而是需要另外一个对象 `KeysAPI`：

```
kapi := etcd.NewKeysAPI(client)
```

所有键值对有关的操作都是这个 `kapi` 对象的方法提供的，它会实现接口的如下方法：

```
type KeysAPI interface {
	// Get 从 etcd 中获取 Node 的信息
	Get(ctx context.Context, key string, opts *GetOptions) (*Response, error)

	// Set 设置对应 key 的值为 value，这个方法也可以用来创建目录
	Set(ctx context.Context, key, value string, opts *SetOptions) (*Response, error)

	// Delete 删除节点，包括目录和文件节点
	Delete(ctx context.Context, key string, opts *DeleteOptions) (*Response, error)

	// Create 是 set 的一种特殊形式，只有之前节点不存在时才会创建成功
	Create(ctx context.Context, key, value string) (*Response, error)

	// CreateInOrder 在目录下面创建递增的键值对
	CreateInOrder(ctx context.Context, dir, value string, opts *CreateInOrderOptions) (*Response, error)

	// Update 也是 Set 的一种特殊形式，只有对应的节点存在时才会更新成功
	Update(ctx context.Context, key, value string) (*Response, error)

	// Watcher 返回一个 Watcher 对象，监听某个节点下面的变化
	Watcher(key string, opts *WatcherOptions) Watcher
}
```

### 创建值

先来看看设置一个 key value 的操作，第一个是 context 对象，后面两个分别是 key 和 value 值，最后是额外的参数，目前并不需要设置为 nil：

```
resp, err := client.Set(context.Background(), "/user/name", "cizixs", nil)
```

如果要为某个值设置超时时间，可以添加选项 `TTL`：

```
resp, err := client.Set(context.Background(), "/user/name", "",
	&etcd.SetOptions{
		TTL: time.Duration(ttl) * time.Second,
		Refresh: true,
	},
)
```

`SetOptions` 可选的字段有：

- `Dir`：布尔值，表示要创建的是一个目录
- `Refresh`：如果设置为 true，则表明只是更新某个 key 的 TTL 时间，不会重置它的值
- `PrevValue`：原子操作，只有节点的值和这个字段指定的值相同时才会执行更新操作
- `PrevIndex`：原子操作，只有节点的 ModifiedIndex 和这个字段指定的值相同时才会执行更新操作
- `PrevExist`：原子操作，只有节点存在或者不存在时才会执行更新操作，支持的值有 `PrevIgnore`、`PrevExist` 和 `PrevNoExist`

这里要先讲一下返回值 `Response` 的构成，它包含了返回中节点的信息以及 index 信息：

```
type Response struct {
	// Action 操作的动作名称，可以是 set、delete
	Action string `json:"action"`

	// Node 代表操作的节点
	Node *Node `json:"node"`

	// PrevNode 节点之前的值
	PrevNode *Node `json:"prevNode"`

	// Index：response 生成是 cluster index 的值
	Index uint64 `json:"-"`
}
```

其中节点的定义如下：

```
type Node struct {
	//  节点的位置，目录名称，比如 `/foo/bar`
	Key string `json:"key"`

	//  节点是否为一个目录
	Dir bool `json:"dir,omitempty"`

	//  节点存储的对象值，如果是目录，则忽略这个字段
	Value string `json:"value"`

	// Nodes 子节点，如果是目录，则这个字段保存了目录下面的所有节点内容
	Nodes Nodes `json:"nodes"`

	//  节点创建时候的 etcd  index
	CreatedIndex uint64 `json:"createdIndex"`

	//  节点更新时候的 etcd index
	ModifiedIndex uint64 `json:"modifiedIndex"`

	// 节点的过期时间
	Expiration *time.Time `json:"expiration,omitempty"`

	// TTL 节点设置的 time to live，单位是秒
	TTL int64 `json:"ttl,omitempty"`
}
```

### 获取值

获取值是通过 `Get` 方法，和 `Set` 一样，它也支持选项。如果只是简单获取某个键的值，直接设置选项为空就行：

```
resp, err := client.Get(context.Background(), key, nil)
```

如果要递归地获取某个目录下面所有的内容，可以使用 `Recursive` 参数：

```
resp, err := client.Get(context.Background(), key, &etcd.GetOptions{
    Recursive: true,   
})
```

然后通过 resp 就能遍历所有的子节点的值。

### 删除值

删除操作通过 `Delete` 方法完成，简单删除一个文件节点，可以忽略参数：

```
_, err := client.Delete(context.Background(), key, nil)
```

如果要删除某个非空的目录，需要 `Recursive` 参数：

```
_, err := client.Delete(context.Background(), key, &etcd.DeleteOptions{
    Dir: true,
    Recursive: true,
})
```

其他字段包括：

- `PrevValue`：只有节点值和给定的值相同时才执行删除操作
- `PrevIndex`：只有节点的 index 和给定的 index 相同时，才执行删除操作

### 监听值

etcd 另外一个重要的功能是监听目录或者文件的变化，`Watcher` 方法会返回一个对象，调用它的 `Next()` 方法会阻塞，一直到监听的对象有变化，它才会返回变化节点的情况：

```
watcher := client.Watcher(key, &etcd.WatcherOptions{
	Recursive: true,
})

for {
	resp, err := watcher.Next(context.Background())
	if err != nil {
		return err
	}
}
```

其中 `WatcherOptions` 接受 `AfterIndex` 和 `Recursive` 两个参数，分别表示才某个 Index 之后开始监听，以及递归监听某个目录下面所有的节点。

完整的 demo 代码在[ github 上](https://github.com/cizixs/etcd-demo/tree/master/v2)，请前往阅读。

## 参考资料

- [etcd client v2 godoc 文档](https://godoc.org/github.com/coreos/etcd/client)
- [etcd client 源码](https://github.com/coreos/etcd/tree/master/client)
- [谈谈CoreOS的etcd](http://dockone.io/article/801)