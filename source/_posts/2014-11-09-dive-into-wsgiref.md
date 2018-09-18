---
layout: post
title: "wsgiref 源码解析"
excerpt: " 看源码是件痛苦的事情。"
categories: 程序技术
tags: [wsgi, wsgiref, python]
comments: true
share: true
---

![](http://mitsuhiko.pocoo.org/wsgi-snake.png)

[图片来源](http://lucumr.pocoo.org/2007/5/21/getting-started-with-wsgi/)

## 介绍
要很好地理解下面的代码，最好有一定的 socket 编程基础，了解 socket 的基本概念和流程。

wsgiref 是 [PEP 333][pep333] 定义的 wsgi 规范的范例实现，里面的功能包括了：

+ 操作 wsgi 的环境变量
+ 应答头部的处理
+ 实现简单的 HTTP server
+ 简单的对程序端和服务器端校验函数

我们先看一个简单的代码实例，然后跟着例子去理解源码：


	
app.py

	
	# pep333 定义的程序端可调用对象
	
	def hello_world_app(environ, start_response):
	    status = '200 OK' # HTTP Status
	    headers = [('Content-type', 'text/plain')] # HTTP Headers
	    start_response(status, headers)
	
	    # The returned object is going to be printed
	    return ["Hello World"]

server.py 
	
	from app import hello_world_app
	from wsgiref.simple_server import make_server
	
	httpd = make_server('', 8000, hello_world_app)
	print "Serving on port 8000..."

	# Serve until process is killed
	httpd.serve_forever()

然后执行 `python server.py` 启动 sever，用 curl 发送一个请求 `curl -i http://localhost:8000/`，会有以下输出：


	HTTP/1.0 200 OK
	Date: Sat, 08 Nov 2014 09:08:05 GMT
	Server: WSGIServer/0.1 Python/2.7.3
	Content-type: text/plain
	Content-Length: 12
	
	Hello World
	
server 的终端会有一条记录：

	Serving on port 8000...
	localhost - - [08/Nov/2014 09:08:05] "GET / HTTP/1.1" 200 12

如何使用就讲到这里，下面就开始源码之旅吧！

 
## 源码分析

你可以使用 `python -c 'import wsgiref; help(wsgiref)'` 查看 wsgiref 库的路径和简介等信息，wsgiref 文件夹的结构如下：

	wsgiref
	|-- handlers.py			# 核心代码，负责 wsgi 程序的处理
	|-- headers.py			# 头部处理的代码
	|-- __init__.py			# 
	|-- simple_server.py	# 简单的 wsgi HTTP 服务器实现
	|-- util.py				# 帮助函数
	`-- validate.py			# wsgi 格式检查和校验

主要的代码结构如下图所示：

![](http://ww2.sinaimg.cn/large/005yyi5Jjw1em4xm3stghj30ps0d5t9l.jpg
)



### simple_server.py
我们先看一下 `make_server` 是怎么启动一个 wsgi 服务器的：


	def make_server(host, port, app, server_class=WSGIServer, handler_class=WSGIRequestHandler):
		server = server_class((host, port), handler_class)
	    server.set_app(app)
	    return server

这个函数做的事情就是：监听在本地的端口上，接受来自客户端的请求，通过 WSGIServer 和 WSGIRequestHandler 处理后，把请求交给程序的的可调用对象 app，然后返回 app 的结果给客户端。

这里有两个重要的类：WSGIServer 和 WSGIRequestHandler。下面分别看一下它们的代码和执行的功能。

	class WSGIServer(HTTPServer):

	    """BaseHTTPServer that implements the Python WSGI protocol"""
	
	    application = None
	
	    def server_bind(self):
	        """Override server_bind to store the server name."""
	        HTTPServer.server_bind(self)
	        self.setup_environ()
	
	    def setup_environ(self):
	        # Set up base environment
	        env = self.base_environ = {}
	        env['SERVER_NAME'] = self.server_name
	        env['GATEWAY_INTERFACE'] = 'CGI/1.1'
	        env['SERVER_PORT'] = str(self.server_port)
	        env['REMOTE_HOST']=''
	        env['CONTENT_LENGTH']=''
	        env['SCRIPT_NAME'] = ''
	
	    def get_app(self):
	        return self.application
	
	    def set_app(self,application):
	        self.application = application

WSGIServer 在原来的 HTTPServer 上面封装了一层，在原来的 HTTPServer 的基础上又额外做了下面的事情：

+ 覆写原来的 server_bind 函数，添加初始化 environ 变量的动作
+ 添加了处理满足 wsgi 的 app 函数：set_app 和 get_app

然后看另外一个类 WSGIRequestHandler：

	class WSGIRequestHandler(BaseHTTPRequestHandler):

	    server_version = "WSGIServer/" + __version__
	
	    def get_environ(self):
	        env = self.server.base_environ.copy()
	        env['SERVER_PROTOCOL'] = self.request_version
	        env['REQUEST_METHOD'] = self.command
	        if '?' in self.path:
	            path,query = self.path.split('?',1)
	        else:
	            path,query = self.path,''
	
	        env['PATH_INFO'] = urllib.unquote(path)
	        env['QUERY_STRING'] = query
	
	        host = self.address_string()
	        if host != self.client_address[0]:
	            env['REMOTE_HOST'] = host
	        env['REMOTE_ADDR'] = self.client_address[0]
	
	        if self.headers.typeheader is None:
	            env['CONTENT_TYPE'] = self.headers.type
	        else:
	            env['CONTENT_TYPE'] = self.headers.typeheader
	
	        length = self.headers.getheader('content-length')
	        if length:
	            env['CONTENT_LENGTH'] = length
	
	        for h in self.headers.headers:
	            k,v = h.split(':',1)
	            k=k.replace('-','_').upper(); v=v.strip()
	            if k in env:
	                continue                    # skip content length, type,etc.
	            if 'HTTP_'+k in env:
	                env['HTTP_'+k] += ','+v     # comma-separate multiple headers
	            else:
	                env['HTTP_'+k] = v
	        return env
	
	    def get_stderr(self):
	        return sys.stderr
	
	    def handle(self):
	        """Handle a single HTTP request"""
	
	        self.raw_requestline = self.rfile.readline()
	        if not self.parse_request(): # An error code has been sent, just exit
	            return
	
	        handler = ServerHandler(
	            self.rfile, self.wfile, self.get_stderr(), self.get_environ()
	        )
	        handler.request_handler = self      # backpointer for logging
	        handler.run(self.server.get_app())

这个类从名字就能知道它的功能——处理客户端的 HTTP 请求，它也是在原来处理 http 请求的BaseHTTPRequestHandler 类上添加了 wsgi 规范相关的内容。

+ get_environ： 解析 environ 变量
+ handle： 处理请求，把封装的环境变量交给 ServerHandler，然后由 ServerHandler 调用 wsgi app，ServerHandler 类会在下面介绍。

### handler.py
这个文件主要是 wsgi server 的处理过程，定义 start_response、调用 wsgi app 、处理 content-length 等等。

可以参考[这篇文章](https://cizixs.github.io/2014/11/08/understand-wsgi/)里的 wsgi server 的简单实现。

![](http://ww2.sinaimg.cn/large/005yyi5Jjw1em4xzexur2j30sr0yk77n.jpg
)


## 一条 HTTP 请求的旅程
服务器端启动服务，等到客户端输入 `curl -i http://localhost:8000/` 命令，摁下回车键，看到终端上的输出，整个过程中，wsgi 的服务器端发生了什么呢？

1. 服务器程序创建 socket，并监听在特定的端口，等待客户端的连接
2. 客户端发送 http 请求
3. socket server 读取请求的数据，交给 http server
4. http server 根据 http 的规范解析请求，然后把请求交给 WSGIServer
5. WSGIServer 把客户端的信息存放在 environ 变量里，然后交给绑定的 handler 处理请求
6. HTTPHandler 解析请求，把 method、path 等放在 environ，然后 WSGIRequestHandler 把服务器端的信息也放到 environ 里
7. WSGIRequestHandler 调用绑定的 wsgi ServerHandler，把上面包含了服务器信息，客户端信息，本次请求信息得 environ 传递过去
8. wsgi ServerHandler 调用注册的 wsgi app，把 environ 和 start_response 传递过去
9. wsgi app 将reponse header、status、body 回传给 wsgi handler
10. 然后 handler 逐层传递，最后把这些信息通过 socket 发送到客户端
11. 客户端的程序接到应答，解析应答，并把结果打印出来。

[pep333]: http://legacy.python.org/dev/peps/pep-0333/
