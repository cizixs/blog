---
layout: post
title: "HTTP 隧道代理原理和实现"
excerpt: "HTTP 隧道代理可以支持 HTTPS 报文的转发，事实上任何 TCP 协议的报文都很通过 HTTP 隧道代理转发。"
categories: blog
tags: [ http, proxy, golang, tunnel]
comments: true
share: true
---

## HTTP 隧道代理简介

在[上一篇文章中](http://cizixs.com/2017/03/21/http-proxy-and-golang-implementation)我们介绍了 HTTP 普通代理的原理和 golang 简单实现，也提到过普通代理的局限，比如无法代理 HTTPS 的报文，此外普通代理也只能代理 HTTP 协议报文，无法支持其他协议的数据转发。当然有问题就有解决方案，对于这些问题 HTTP 可以通过隧道（tunnel）代理来解决。

隧道代理的原因也可以用一句话来总结：

> 代理服务器和真正的服务器之间建立起 TCP 连接，然后在客户端和真正服务器端进行数据的直接转发。


《HTTP 权威指南》对隧道描述的原话是：

> The CONNECT method asks a tunnel gateway to create a TCP connection to an arbitrary destination server and port and to blindly relay subsequent data between client and server.

意思和我之前的那句话一样，只不过有了更多的细节说明。

下图是《HTTP 权威指南》书中的插图，它讲解了客户端通过隧道代理连接 HTTPS 服务器的过程。

- （a）客户端先发送 CONNECT 请求到隧道代理服务器，告诉它建立和服务器的 TCP 连接（因为是 TCP 连接，只需要 ip 和端口就行，不需要关注上层的协议类型）
- （b，c）代理服务器成功和后端服务器建立 TCP 连接
- （d）代理服务器返回 `HTTP 200 Connection Established` 报文，告诉客户端连接已经成功建立
- （e）这个时候就建立起了连接，所有发给代理的 TCP 报文都会直接转发，从而实现服务器和客户端的通信

![](https://st.imququ.com/i/webp/static/uploads/2015/11/web_tunnel.png.webp)

### CONNECT 请求

`CONNECT` 请求的内容和其他 HTTP 方法的语法一样，只不过它在状态栏（status line）指定了真正服务器的地址。请求 URI 替换成了 hostname 和 port 的字符串，比如：

```bash
CONNECT realserver.com:443 HTTP/1.0
User-Agent: GoProxy
```

而其他 HTTP 请求的状态栏对应位置是路径地址，比如：

```bash
GET /about HTTP/1.0
User-Agent: GoProxy
```

知道了 hostname 和 port，代理服务器就能正确地建立，才能够继续后面的访问。需要注意的是，客户端应该尽量少地暴露其他信息，最好只有状态栏一行的内容，因为 `CONNECT` 请求是没有经过加密的。如果想通过这种方式进行 HTTPS 安全访问，那么不要在 `CONNECT` 请求中暴露敏感数据（比如 cookie）是必须的。

如果代理服务器正确接受了 `CONNECT` 请求，并且成功建立了和后端服务器的 TCP 连接，它应该返回 `200` 状态码的应答，按照大多数的约定为 `200 Connection Establised`。应答也不需要包含其他的头部和 body，因为后续的数据传输都是直接转发的，代理不会分析其中的内容。


## 代码实现

有了上面的理论分析，我们可以写出下面的代码：

```golang
package main

import (
	"fmt"
	"io"
	"net"
	"net/http"
)

type Pxy struct {}

func NewProxy() *Pxy {
	return &Pxy{}
}

// ServeHTTP is the main handler for all requests.
func (p *Pxy) ServeHTTP(rw http.ResponseWriter, req *http.Request) {
	fmt.Printf("Received request %s %s %s\n",
		req.Method,
		req.Host,
		req.RemoteAddr,
	)

	if req.Method != "CONNECT" {
		rw.WriteHeader(http.StatusMethodNotAllowed)
		rw.Write([]byte("This is a http tunnel proxy, only CONNECT method is allowed."))
		return
	}

    // Step 1
	host := req.URL.Host
	hij, ok := rw.(http.Hijacker)
	if !ok {
		panic("HTTP Server does not support hijacking")
	}

	client, _, err := hij.Hijack()
	if err != nil {
		return
	}

    // Step 2
	server, err := net.Dial("tcp", host)
	if err != nil {
		return
	}
	client.Write([]byte("HTTP/1.0 200 Connection Established\r\n\r\n"))

    // Step 3
	go io.Copy(server, client)
	io.Copy(client, server)
}

func main() {
	proxy := NewProxy()
	http.ListenAndServe("0.0.0.0:8080", proxy
}

```

这段代码和前一篇文章的大致框架相同，但是处理的逻辑有些许的区别。

1. 首先接收到客户端的请求之后，代理服务器需要获得底层的 TCP 连接，这样才能转发数据，所以这里看到了 `Hijacker` 类型转换和 `Hijack()` 调用，它们最终的目的是拿到客户端的 TCP 连接（`net.TCPConn`）
2. 然后代理服务器调用 `net.Dial` 函数和真正的服务器端建立 TCP 连接，并在成功后返回给客户端 `200` 应答
3. 最后就是在客户端和服务器端直接转发数据

**NOTE**：[默认的 ServeMux 不支持 `CONNECT` 方法的请求](https://github.com/golang/go/issues/9561)，请直接使用自己编写的 `Proxy` 作为 Mux。

稍加修改我们就能实现同时支持两种代理的代码，我已经把最终的代码放到了 [github 上面](https://github.com/cizixs/pxy)。

## 参考资料

- [HTTP 代理原理及实现（一）](https://imququ.com/post/web-proxy.html)
- [一个简单的Golang实现的HTTP Proxy](http://www.flysnow.org/2016/12/24/golang-http
