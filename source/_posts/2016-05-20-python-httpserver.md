---
layout: post
title: "Python BaseHTTPServer 介绍"
excerpt: "介绍 Python 自带的 HTTP Server，编写一个极简的 Todo API。虽然编写 Web 应用的时候直接使用框架，但是了解这些底层的基础库总没有什么害处。"
categories: blog
tags: [python, http, server, library]
comments: true
share: true
---

本文针对 python 2.7 版本，介绍了 BaseHTTPServer 这个库的使用方法。

这个库是 python 自带的标准库的一部分，不需要额外安装，在 linux 系统下，位置在 `/usr/lib/python2.7/BaseHTTPServer.py`。

## HTTP 协议

### HTTP 请求（request）

http 请求分为三个部分：

1. 第一行：请求类型、地址和版本号
2. 头部信息：HTTP header
3. 数据部分

标准的 HTTP 请求是：
```
GET / HTTP/1.1
Host: cizixs.com
User-Agent: Mozilla/5.0 (Windows NT 6.1; WOW64; rv:42.0) Gecko/20100101 Firefox/42.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
Connection: keep-alive
If-Modified-Since: Thu, 25 Feb 2016 16:00:57 GMT
Cache-Control: max-age=0
```

标准的 HTTP 响应头部：
```
HTTP/1.1 304 Not Modified
Server: GitHub.com
Date: Thu, 24 Mar 2016 06:21:25 GMT
Last-Modified: Thu, 25 Feb 2016 16:00:57 GMT
access-control-allow-origin: *
Expires: Thu, 24 Mar 2016 06:31:25 GMT
Cache-Control: max-age=600
X-GitHub-Request-Id: 3AF60A59:7CE3:1C889201:56F38765

data...
```

## 使用 BaseHTTPServer 写一个简单的 web server
这个类可以帮助你快速编写一个 HTTP 服务端 server。别说了，先上代码：

    from BaseHTTPServer import BaseHTTPRequestHandler
    import cgi
    import json


    class TodoHandler(BaseHTTPRequestHandler):
        """A simple TODO server

        which can display and manage todos for you.
        """

        # Global instance to store todos. You should use a database in reality.
        TODOS = []

        def do_GET(self):
            # return all todos

            if self.path != '/':
                self.send_error(404, "File not found.")
                return

            # Just dump data to json, and return it
            message = json.dumps(self.TODOS)

            self.send_response(200)
            self.send_header('Content-type', 'application/json')
            self.end_headers()
            self.wfile.write(message)

        def do_POST(self):
            """Add a new todo

            Only json data is supported, otherwise send a 415 response back.
            Append new todo to class variable, and it will be displayed
            in following get request
            """
            ctype, pdict = cgi.parse_header(self.headers['content-type'])
            if ctype == 'application/json':
                length = int(self.headers['content-length'])
                post_values = json.loads(self.rfile.read(length))
                self.TODOS.append(post_values)
            else:
                self.send_error(415, "Only json data is supported.")
                return

            self.send_response(200)
            self.send_header('Content-type', 'application/json')
            self.end_headers()

            self.wfile.write(post_values)

    if __name__ == '__main__':
        # Start a simple server, and loop forever
        from BaseHTTPServer import HTTPServer
        server = HTTPServer(('localhost', 8888), TodoHandler)
        print("Starting server, use <Ctrl-C> to stop")
        server.serve_forever()

这段代码实现的功能很简单，就是一个简单的 Todo 管理：你可以添加 todo，也可以查询 todo。更新和删除 todo 可以根据上面的代码自行添加。

代码也不难理解，在关键的步骤我已经添加了注释，在这里就不再解释了。

好了，我们用 [httpie]() 来和它交互一下，最开始的时候返回的数据是空的

```
➜  ~ http --verbose http://localhost:8888
GET / HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate
Host: localhost:8888
User-Agent: HTTPie/0.8.0


HTTP/1.0 200 OK
Content-type: application/json
Date: Fri, 25 Mar 2016 09:35:08 GMT
Server: BaseHTTP/0.3 Python/2.7.10

[]
```

然后，添加几条试试：

```
➜  ~ http --verbose POST http://localhost:8888 content="buy a beer" finished:=false
POST / HTTP/1.1
Accept: application/json
Accept-Encoding: gzip, deflate
Content-Length: 44
Content-Type: application/json; charset=utf-8
Host: localhost:8888
User-Agent: HTTPie/0.8.0

{
    "content": "buy a beer",
    "finished": false
}

HTTP/1.0 200 OK
Content-type: application/json
Date: Fri, 25 Mar 2016 09:36:08 GMT
Server: BaseHTTP/0.3 Python/2.7.10

{u'content': u'buy a beer', u'finished': False}

➜  ~ http --verbose POST http://localhost:8888 content="learn HTTP" finished:=false
POST / HTTP/1.1
Accept: application/json
Accept-Encoding: gzip, deflate
Content-Length: 44
Content-Type: application/json; charset=utf-8
Host: localhost:8888
User-Agent: HTTPie/0.8.0

{
    "content": "learn HTTP",
    "finished": false
}

HTTP/1.0 200 OK
Content-type: application/json
Date: Fri, 25 Mar 2016 09:36:24 GMT
Server: BaseHTTP/0.3 Python/2.7.10

{u'content': u'learn HTTP', u'finished': False}
```

这个时候，再来看一下内容：

```
➜  ~ http --verbose http://localhost:8888
GET / HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate
Host: localhost:8888
User-Agent: HTTPie/0.8.0


HTTP/1.0 200 OK
Content-type: application/json
Date: Fri, 25 Mar 2016 09:36:58 GMT
Server: BaseHTTP/0.3 Python/2.7.10

[
    {
        "content": "buy a beer",
        "finished": false
    },
    {
        "content": "learn HTTP",
        "finished": false
    }
]
```
我们刚刚创建的 todo 就出现了！前面也说了，这段代码不支持更新和删除功能，而且数据也没有落地，关闭程序之后，数据就消失了。

## BaseHTTPServer 源代码解析

BaseHTTPServer 这个模块提供了两个类让开发者实现 HTTP server：`HTTPServer` 和 `BaseHTTPRequestHandler`。

`HTTPServer` 继承了 `SocketServer.BaseServer`，主要功能是：创建和监听 socket，把请求转发给 handler 去处理。主要的工作都是在 `BaseHTTPRequestHandler` 中处理的，它把和请求有关的信息都封装成自己的实例变量，可以在子类中直接使用。这些变量包括：

+ client_address：客户端的地址，存放在一个 tuple 里 (host, port)
+ server: server 实例
+ command：请求类型，比如，GET、POST 等
+ path：请求路径，比如 `/index.html`
+ request_version: 请求版本号，比如 `HTTP/1.0`
+ headers：`mimetools.Message` 的实例对象，包含了头部信息
+ rfile：rfile 是一个输入流，用来读取请求的数据
+ wfile：wfile 是一个输出流，用来回写响应，回写的数据必须遵守 HTTP 协议的格式

除了这些实例变量之外，还有其他的类变量：

+ server_version：服务器的版本号，比如 `BaseHTTP/0.2`
+ sys_version：python 的版本号，比如 `Python/1.4`
+ error_message_format：错误 response 的格式
+ error_content_type：错误 response 的 Content-Type，默认是 `text/html`
+ protocol_version：HTTP 协议版本号
+ responses：error code 对应错误消息的匹配关系

当然，还有一些方法可以使用：

+ handle()：调用底层的实现来处理一次请求
+ send_response（）：发送应答消息和状态码

更多的内容可以查看文末的链接。

## 为什么这个类不会被广泛使用？

写了这么多，我们也看到这个类缺点很多，比如：

+ 不支持 url 解析和转发，如果有多个 endpoint，需要用户自己解析
+ 回写的响应也需要用户自己维护格式，容易出错
+ 没有模板支持，如果要写 HTML 页面，也需要自己维护

所以在正式的工作中，编写 HTTP server 端应用的时候，都是使用 web 框架的。因为 web 框架帮你封装了底层的这些细节，还提供了很多便利的功能，让开发者把中心更多地放到业务逻辑的实现。

如果有时间，以后讲讲怎么自己写一个简单的 HTTP Server。

## 参考资料

+ [BaseHTTPServer — Basic HTTP server](https://docs.python.org/2/library/basehttpserver.html)
