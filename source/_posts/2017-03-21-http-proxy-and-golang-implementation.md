---
layout: post
title: "HTTP 代理原理和实现"
excerpt: "代理的核心功能可以用一句话概括：接受客户端的请求，转发到后端服务器，获得应答之后返回给客户端。"
categories: blog
tags: [http, proxy, golang]
comments: true
share: true
---

代理的核心功能可以用一句话概括：接受客户端的请求，转发到后端服务器，获得应答之后返回给客户端。下图是 《HTTP 权威指南》一书中给出的图例，可以很清晰地说明这一流程：

![](https://st.imququ.com/i/webp/static/uploads/2015/11/web_proxy.png.webp)

代理的功能有很多，事实上整个互联网到处都充斥着代理服务器。如果所有的 HTTP 访问都是客户端和服务器端直接进行的话，我们的网络不仅会变得缓慢，而且性能会大打折扣。

代理服务器根据不同的配置和使用，可能会有不同的功能，这些功能主要包括：

- 内容过滤：代理可以根据一定的规则限制某些请求的连接。比如有些公司会设置内部网络无法访问某些购物、游戏网站，或者学校的网络不让学生访问色情暴力的网站等
- 节省成本：代理服务器可以作为缓存使用，对于某些资源只需要第一次访问的时候去下载，以后代理直接把缓存的结果返回给客户端，节约网络带宽的开销
- 提高性能：通过代理服务器的缓存（比如 CDN）和负载均衡（比如 nginx lb）功能，服务器端可以加速请求的访问，在更快的时间内返回结果）
- 增加安全性：公司可以在内网和外网之间通过代理进行转发，这样不仅对外隐藏了实现的细节，而且可以在代理层对爬虫、病毒性请求进行过滤，保护内部服务

所有的这些功能的实现都依赖于代理的特性，它可以在客户端和服务器端做一些事情，根据代理做的事情不同，它的角色和功能也就不同。那么，代理具体可以做哪些事情呢？比如：

- 修改 HTTP 请求：url、header、body
- 过滤请求：根据一定的规则丢弃、过滤请求
- 决定转发到哪个后端（可以是静态定义的，也可以是动态决定）
- 保存服务器的应答，后续的请求可以直接使用保存的应答
- 修改应答：对应答做一些格式的转换，修改数据，甚至返回完全不一样的应答数据
- 重试机制，如果后端服务器暂时无法响应，隔一段时间重试
- ……

## 正向代理和反向代理

代理可以分为正向代理和反向代理两种。

正向代理需要客户端来配置，一般来说我们会通过浏览器或者操作系统提供的工具或者界面来配置。这个时候，代理对客户端不是透明的，客户端需要知道代理的地址并且手动配置。配置了代理，浏览器在发送请求的时候会对报文做特殊的修改。

反向代理对客户端是透明的，也就是说客户端一般不知道代理的存在，认为自己是直接和服务器通信。我们大部分访问的网站就是反向代理服务器，反向代理服务器会转发到真正的服务器，一般在反向代理这一层实现负载均衡和高可用的功能。而且这里也可以看到，客户端是不会知道真正服务器端的 ip 地址和端口的，这在一定程度上起到了安全保护的作用。

### 代理服务器怎么知道目的服务器的地址？

在反向代理中，代理服务器要转发的服务器地址都是事先知道的（包括静态配置和动态配置）。比如[使用 nginx 来配置负载均衡][1]。

而对于正向代理来说，客户端可能访问的服务器地址是无法事先知道的。因为HTTP 协议活动在应用层，它无法获取网络层（IP层）信息，那么该协议要有一个地方可以拿到这个信息。HTTP 中可能保存这个信息的地方有两个：URL 和 header。默认情况下，HTTP 请求的 status line 有三部分组成：方法、uri 和协议版本，比如：

```
GET /index.html HTTP/1.0
User-Agent: gohttp 1.0
```

如果客户端（比如浏览器）知道自己在通过正向代理进行报文传输，那么它会在 status line 加上要访问服务器的真实地址。这个时候发送的报文是：

```
GET http://www.marys-antiques.com/index.html HTTP/1.0
User-Agent: gohttp 1.0
```

### 代理路径

客户端不管是通过代理服务器，还是直接访问后端服务器对于最终的结果是没有区别的，也就是说大多数情况下客户端根本不关心它访问的到底是什么，只需要（准确快速地）拿到想要的信息就够了。但是有时候，我们还是希望知道请求到底在中间经历了哪些代理，比如用来调试网络异常，或者做数据统计，而 HTTP 协议也提供了响应的功能。

虽然 RFC 2616 定义了 `Via` 头部字段来跟踪 HTTP 请求经过的代理路径，但在实际中用的更多的还是 [`X-Forwarded-For`](https://en.wikipedia.org/wiki/X-Forwarded-For) 字段，`X-Forwarded-For` 是 `Squid` 缓存代理服务软件引入的，目前已经在规范化在 [RFC 7239](https://tools.ietf.org/html/rfc7239) 文档。

`X-Forwarded-For` 头部格式也比较简单，比如某个服务器接受到请求的对应头部可能是：

```bash
X-Forwarded-For: client, proxy1, proxy2
```

对应的值有多个字段，每个字段代表中间的一个节点，它们之间由逗号和空格隔开，从左到右距离当前节点越来越近。

每个代理服务器会在 `X-Forwarded-For` 头部填上前一个节点的 ip 地址，这个地址可以通过 TCP 请求的 remote address 获取。为什么每个代理服务器不填写自己的 ip 地址呢？有两个原因，如果由代理服务器填写自己的 ip 地址，那么代理可以很简单地伪造这个地址，而上一个节点的 remote address 是根据 TCP 连接获取的（如果不建立正确的 TCP 连接是无法进行 HTTP 通信的）；另外一个原因是如果由当前节点填写 `X-Forwarded-For`，那么很多情况客户端无法判断自己是否会通过代理的。

**NOTE**：
1. 最终客户端或者服务器端接受的请求，`X-Forwarded-For` 是没有最邻近节点的 ip 地址的，而这个地址可以通过 remote address 获取
2. 每个节点（不管是客户端、代理服务器、真实服务器）都可以随便更改 `X-Forwarded-For` 的值，因此这个字段只能作为参考

## 代理服务器实现

这个部分我们会介绍如何用 golang 来实现 HTTP 代理服务器，需要读者了解一些 HTTP 服务器端编程的知识，可以参考我之前的文章：[go http 服务器编程](http://cizixs.com/2016/08/17/golang-http-server-side)。

### 正向代理

按照我们之前介绍的代理原理，我们可以编写出这样的代码：

```go
package main

import (
	"fmt"
	"io"
	"net"
	"net/http"
	"strings"
)

type Pxy struct {}

func (p *Pxy) ServeHTTP(rw http.ResponseWriter, req *http.Request) {
	fmt.Printf("Received request %s %s %s\n", req.Method, req.Host, req.RemoteAddr)

	transport :=  http.DefaultTransport

	// step 1
	outReq := new(http.Request)
	*outReq = *req // this only does shallow copies of maps

	if clientIP, _, err := net.SplitHostPort(req.RemoteAddr); err == nil {
		if prior, ok := outReq.Header["X-Forwarded-For"]; ok {
			clientIP = strings.Join(prior, ", ") + ", " + clientIP
		}
		outReq.Header.Set("X-Forwarded-For", clientIP)
	}

	// step 2
	res, err := transport.RoundTrip(outReq)
	if err != nil {
		rw.WriteHeader(http.StatusBadGateway)
		return
	}

	// step 3
	for key, value := range res.Header {
		for _, v := range value {
			rw.Header().Add(key, v)
		}
	}

	rw.WriteHeader(res.StatusCode)
	io.Copy(rw, res.Body)
	res.Body.Close()
}

func main() {
	fmt.Println("Serve on :8080")
	http.Handle("/", &Pxy{})
	http.ListenAndServe("0.0.0.0:8080", nil)
}
```

这段代码比较直观，只包含了最核心的代码逻辑，完全按照最上面的代理图例进行组织。一共分成几个步骤：

1. 代理接收到客户端的请求，复制了原来的请求对象，并根据数据配置新请求的各种参数（添加上 `X-Forward-For` 头部等）
2. 把新请求发送到服务器端，并接收到服务器端返回的响应
3. 代理服务器对响应做一些处理，然后返回给客户端

上面的代码运行之后，会在本地的 `8080` 端口启动代理服务。修改浏览器的代理为 `127.0.0.1：:8080` 再访问网站，可以验证代理正常工作，也能看到它在终端打印出所有的请求信息。

虽然这段代码非常简短，但是你可以添加更多的逻辑实现非常有用的功能。比如在请求发送之前进行过滤，根据一定的规则直接阻止某些请求的访问；或者对请求进行限流，某个客户端在一定的时间里执行的请求有最大限额；统计请求的数据进行分析等等。

这个代理目前不支持 HTTPS 协议，因为它只提供了 HTTP 请求的转发功能，并没有处理证书和认证有关的内容。如果了解 HTTPS 协议的话，你会明白这种模式下是无法完成 HTTPS 握手的，虽然代理可以和真正的服务器建立连接（知道了对方的公钥和证书），但是代理无法代表服务器和客户端建立连接，因为代理服务器无法知道真正服务器的私钥。

### 反向代理

编写反向代理按照上面的思路当然没有问题，只需要在第二步的时候，根据之前的配置修改 `outReq` 的 URL Host 地址可以了。不过 Golang 已经给我们提供了编写代理的框架：[`httputil.ReverseProxy`](https://golang.org/pkg/net/http/httputil/#ReverseProxy)。我们可以用非常简短的代码来实现自己的代理，而且内部的细节问题都已经被很好地处理了。

这部分我们会实现一个简单的反向代理，它能够对请求实现负载均衡，随机地把请求发送给某些配置好的后端服务器。使用 `httputil.ReverseProxy` 编写反向代理最重要的就是实现自己的 `Director` 对象，这是 `GoDoc` 对它的介绍：

> Director must be a function which modifies the request into a new request to be sent using Transport.
> Its response is then copied back to the original client unmodified.
> Director must not access the provided Request after returning.

简单翻译的话，`Director` 是一个函数，它接受一个请求作为参数，然后对其进行修改。修改后的请求会实际发送给服务器端，因此我们编写自己的 `Director` 函数，每次把请求的 Scheme 和 Host 修改成某个后端服务器的地址，就能实现负载均衡的效果（其实上面的正向代理也可以通过相同的方法实现）。看代码：

```go
package main

import (
        "log"
        "math/rand"
        "net/http"
        "net/http/httputil"
        "net/url"
)

func NewMultipleHostsReverseProxy(targets []*url.URL) *httputil.ReverseProxy {
        director := func(req *http.Request) {
                target := targets[rand.Int()%len(targets)]
                req.URL.Scheme = target.Scheme
                req.URL.Host = target.Host
                req.URL.Path = target.Path
        }
        return &httputil.ReverseProxy{Director: director}
}

func main() {
        proxy := NewMultipleHostsReverseProxy([]*url.URL{
                {
                        Scheme: "http",
                        Host:   "localhost:9091",
                },
                {
                        Scheme: "http",
                        Host:   "localhost:9092",
                },
        })
        log.Fatal(http.ListenAndServe(":9090", proxy))
}
```

NOTE：这段代码来自 http://blog.charmes.net/2015/07/reverse-proxy-in-go.html。

我们让代理监听在 `9090` 端口，在后端启动两个返回不同响应的服务器分别监听在 `9091` 和 `9092` 端口，通过 `curl` 访问，可以看到多次请求会返回不同的结果。

```bash
➜  curl http://127.0.0.1:9090
116064a9eb83
➜  curl http://127.0.0.1:9090
8f7ccc11718f
```

同样的，这段代码也只是一个 demo，存在着很多问题，比如没有错误处理机制，如果后端某个服务器挂了，代理会返回 502 错误，更好的做法是把请求转发到另外的可用服务器。当然也可以添加更多的特性让它更好用，比如动态地添加后端服务器列表；根据后端服务器的负载情况进行负载转发等等。

## 参考资料

- [Reverse Proxy in Go](http://blog.charmes.net/2015/07/reverse-proxy-in-go.html)
- [Go lang simple reverse proxy](http://www.darul.io/post/2015-07-22_go-lang-simple-reverse-proxy)
- [http 权威指南](http://shop.oreilly.com/product/9781565925090.do)

  [1]: http://nginx.org/en/docs/http/load_balancing.html
