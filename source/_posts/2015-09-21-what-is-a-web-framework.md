---
layout: post
title: "【翻译】什么是 web 框架？"
excerpt: "使用不同的 web 框架写了不少代码，深入了解一下 web 框架工作原理自然会有好处。"
categories: blog
tags: [python, web, framework]
comments: true
share: true
---

原文地址： http://jeffknupp.com/blog/2014/03/03/what-is-a-web-framework/

Web 应用框架，简称为 `web 框架`，是编写 web 应用程序的基石。不管简单的博客系统，还是 Ajax 为主的应用，网络上所有的页面都是代码构成的。进来我发现，很多想学习诸如 Flask 或者 Django 等 web 框架的开发者，并不很了解 web 框架是什么，它们的作用和工作原理。这篇文章，我将会讲一下这个通常会被忽略的话题。希望读完这篇文章，你能比较深刻地理解 web 框架到底是什么，还有为什么会有 web 框架。这些知识将有利于你学习新的 web 框架，而且在选择 web 框架的时候有法可依。

## WEB 是什么工作的？

在讨论框架之前，我们要先了解一下网页是怎么工作。我们就从你在浏览器输入一个网址，摁下 enter 键说起。打开你的浏览器，输入 `http://jeffknupp.com`（译者注：原作者的个人网站首页），我们来看看你的浏览器做了那些事情（DNS 查询的 buffer 就略过），才能显示你看到的网页。

### web servers，和 web ... servers ...

浏览器接接收到的网页都是 `HTML` 文件，`HTML` 是一种描述网页内容和结构的语言。负责给浏览器发送 `HTML` 的程序称为 web server，容易混淆的是，这个应用程序所在的机器通常也被称为 web server。

最重要的一点是，所有的 web 应用做的事情就是把 `HTML` 内容发送给浏览器。不论这个 web 应用有多么复杂，最终的任务都是把 `HTML`（我故意忽略掉其他格式的内容，比如 JSON，CSS 文件，因为原理都是一样的）发送给浏览器。

问题来了：web 应用如何知道要发送什么内容给浏览器呢？**答案：它会发送浏览器请求的内容。**

### HTTP

浏览器从 web server 下载内容所用的是 `HTTP` 协议（协议在计算机科学中，指的是双方通信所共同遵循的数据格式和通信步骤）。`HTTP` 协议的基础是 `请求-应答` (`request-response`) 模型。客户端（你的浏览器）`请求
` 某台物理机上 web 应用的数据，web server 则负责 `应答`请求的数据。

有个重要的事情是：所有的通信都是客户端（你的浏览器）发起的。服务端（web server）是不可能主动连接你，发送没有请求的数据的。如果你收到了数据，只是因为你的浏览器主动请求了这些数据。

#### HTTP 方法

`HTTP` 协议的每条消息都有对应的方法（method），不同的方法对应了客户端能发起的不同请求，也对应了客户端不同的意图。比如，*请求* 网页的 HTML 和提交一个表格在逻辑上是不同的，所以这两种方法需要两种不同的方法。

#### HTTP GET

顾名思义，`GET` 方法就是从 web server 获取（get）数据，`GET` 请求也是目前最常用的 `HTTP` 请求。 处理 `GET` 请求的过程中，web 应用只需要返回请求的数据，无需其他操作。尤其是，不应该修改应用的状态（比如， `GET` 请求不应该导致一个新用户被创建）。因为这个原因，`GET` 请求通常被看做是 `安全` 的。

#### HTTP POST

和网站的交互，明显不只是查看网页的。我们还会通过表格等形式发送数据给 web 应用，这些操作需要用到另外一种请求：`POST`。`POST` 请求通常会传递用户创建的信息，导致 web 应用执行某些动作。输入自己的信息，来注册某个网站就会用到 `POST` 请求，请求中会包含你输入的数据。

和 `GET` 请求不同的是， `POST` 请求通常会导致 web 应用状态的改变。上面提及的例子中，表单被提交后，一个新的用户会被创建。还有一点不同，`POST` 请求的结果可能不会返回 HTML 数据给客户端，客户端需要通过 `response code` 来判断操作是否成功。

#### HTTP response code

正常情况下，web server 会返回 200 的 response code，意思是：我已经完成了你要我做的事情，并且一切都没有问题。`response code` 是三位的数字，每次应答都要包含一个 response code，来标识请求的结果。`200` 表示 OK，是 `GET` 方法常见的返回值。`POST` 请求经常会返还 `204`(No contnet)，表示：一切正常，但是我没有数据可以展示给你。

还需要注意的是：`POST` 请求发送给的 url，可能和数据发送出去的 url 不同。继续以我们的注册页面为例，注册表可能位于 `http://foo.com/signup`，点击 `submit` 之后，包含着注册数据的 `POST` 请求可能被发送到 `http://foo.com/process_signup`。`POST` 请求要发送到的地址，一般在注册表格的 `HTML` 源码里指定。

### Web 应用

掌握 `GET` 和 `POST` 方法就能做很多事情，因为它们是 web 上最常用的两个方法。总结一下，web 应用就是接收 `HTTP` 请求，然后返回 `HTTP` 应答，一般是包含请求数据的 `HTML`。`POST` 方法会导致 web 应用执行某些动作，例如在数据库添加一条记录。当然还有其他的 `HTTP` 方法，但目前我们只需要关心 `GET` 和 `POST` 就足够啦。

最简单的 web 应用长什么样呢？我们就来写一个监听在 `80` 端口的 web 应用，一旦和客户端建立连接，就等待客户端发起请求，并返回非常简单的 `HTML`。

这个程序是这样的：

    import socket

    HOST = ''
    PORT = 80
    listen_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    listen_socket.bind((HOST, PORT))
    listen_socket.listen(1)
    connection, address = listen_socket.accept()
    request = connection.recv(1024)
    connection.sendall("""HTTP/1.1 200 OK
    Content-type: text/html


    <html>
        <body>
            <h1>Hello, World!</h1>
        </body>
    </html>""")
    connection.close()

（如果上面的程序报端口错误，可以把 `PORT` 的值修改成其他值，比如 `8080`。）

上面的代码只会接收一个连接和一个请求，不管请求的 URL 是什么，都会返回同样的 `HTTP` 内容，response code 是 `200`。（很明显，这不算真正的 web server）。在这个例子，我们告诉客户端，返回的数据格式为 `HTML`，而不是其他的格式，比如 `JSON`。

#### request 请求解析

如果看一下上面例子中发送的 `HTTP` 请求（译者注：可以使用 chrome 的 inspect elements -> Network，或者抓包工具 tcpdump 等工具查看发送的 HTTP 请求），就会发现它和应答很相似。请求的第一行是：

    <HTTP Method> <URL> <HTTP version>

在这个例子里就是 `GET / HTTP/1.1`。第一行后面跟着的是请求的头部（headers），比如 `Accept: */*`（表示可以接收任何格式的内容作为应答）。基本上就这么多。

我们发送的应答也是类似的格式，第一行是:

    <HTTP version> <HTTP Status-Code> <Status-Code Reason-Phrase>

这个例子中就是 `HTTP/1.1 200 OK`。然后是头部，和请求的头部一样。最后是应答的实际内容。注意：应答也可能是字符串或者二进制对象，头部的 `Content-typ` 就是来标识应答内容，让客户端来解析的。

#### web server 更多琐碎的细节

要在上面的例子基础上继续扩展的话，还有很多需要我们来解决的问题：

1. 怎么查看请求的 URL，然后返回不同的页面？
2. 除了 `GET` 请求，怎么处理 `POST` 请求？
3. 怎么处理很复杂的概念，比如 sessions 和 cookies？
4. 如何扩展这个应用，让它可以同时处理数千条连接？

可以想象，没人会愿意每次编写 web 应用的时候都要自己处理这些问题。为了解决这个难题，就会存在很多的软件包帮你处理这些烦人的细节，让开发者可以把心思放到业务逻辑上。记住，不管 web 框架多么负责，其最核心的功能和我们上面的例子是一样的：监听客户端请求，然后返回 `HTML` 给客户端。

NOTE：客户端的框架和上面的内容迥然不同。

### 解决两个难题：路由（routing）和模板（templates）

在构建 web 应用的所有的问题中，有两个比较突出：

1. 怎么把请求 URL 和处理它的那部分代码对应起来？
2. 怎么动态地生产请求内容？包括所有要计算的值，和从数据库获取的信息？

每个 web 框架解决这两个问题的方法都不太相同，我们就举 Flask 和 Django 的例子来说明这个问题。首先，我们还要来说一下 `MVC` 模式。

### Django 中的 MVC

Django 采用 MVC 模式， 所以要求使用这个框架的代码都遵循这个模式。*MVC*，是 *Model-View-Controller* 的缩写，用来分离应用的不同责任。数据库表所代表的资源用 *models* 来表示，*controllers* 负责应用的业务逻辑和操作 models。*Views* 则负责动态生成代表页面的 `HTML`。

不过容易让人混淆的是，django 中 *controllers* 被称作 *views*，而 *views* 被称为 *templates*。除了命名外，django 算是比较直接的 *MVC* 架构。

### Django 的路由（routing）机制

这里说的路由（routing）就是把请求的 URL 对应到处理生成相关 `HTML` 的代码。最简单的例子，所有的请求都是相同的代码处理（就是我们之前编写的代码）。复杂一点呢，每个的 URL 都对应一个不同的 `view function`。比如，有个地方的逻辑是接收到 `www.foo.com/bar` 请求，就把它交给 `handle_bar()` 函数处理。我们可以这样依次编写出所有 url 对应的处理函数。

不过，这个方法有个致命伤：没有办法处理带有动态数据的 URL，比如说资源的 ID（例如 `http://www.foo.com/users/3`）。我们怎么把这个 URL 映射到函数，同时能传过去用户 ID 信息呢？

Django 采用的方法是利用正则表达式：用正则表达式匹配 URL，然后把匹配的数据作为参数传递给处理函数。比如，我可以说匹配 `^/users/(?P<id>\d+)/$` 的 URL 会调用 `display_user(id)` 函数，其中 `id` 就是正则表达式括号里匹配的内容。利用这种方式，任何 `/users/<some number>` 类型的 URL 都能对应到 `display_user` 函数，并且正则表达式可以无限复杂，包含任意的关键字和未知参数。

### Flask 的 路由机制

Flask 采用的是另外一种方法。把 url 对应到函数参照的是 `route()` 装饰器。下面的 Flask 代码和上面提到的正则表达式代码功能相同：

    @app.route('/users/<id:int>/')
    def display_user(id):
        # ...

如你所见，装饰器使用的是简化版的正则表达式来传递参数，参数被 route 参数中 `<name:type>` 的指令捕获。要路由 `/info/about.html` 这样的页面，就需要 `@app.route('/info/about_us.html')`。

### 根据模板生成 HTML

继续上面的例子，一旦我们知道怎么把 URL 对应到逻辑代码，那么要怎么动态地生成 HTML，并且方便开发者手动编辑呢？Django 和 Flask 两者这次方法一样，那就是 —— HTML 模板。

`HTML 模板` 有点像 `string.format()`：预期的输出首先要用站位标识，然后再填入动态的数据。可以把这个网页想象成一个字符串，里面用括号标识动态的数据，最后调用 `str.format()` 生成最终的结果。Django 的 模板引擎和 Flask 采用的 *jinja2* 都是这个原理。

不过，并不是所有的模板引擎地位都一样。Django 的模板只支持简单的变成，而 Jinja2 却能让你执行任意的代码（当然并发完全可以，不过已经很近似）。Jinja2 很会 cache 渲染的结果，下次有同样的参数传过来的时候，就会直接从 cache 获取结果，而不需要重新渲染。

### 数据库集成

Django，宣称“自带电池”（batteries included），然后也会包含 `ORM`（Object Relational Mapper）。`ORM` 的目的有两个：把 python 的类映射到数据的表结构，和通过封装隐藏不同数据库之间的差异（第一点是它更主要的功能）。没有人喜欢 `ORM`（因为不同域之间的 mapping 从不完美），不过这些缺点都是可以接受的。 Django 功能比较全面，Flask 作为一个微框架，并不自带 `ORM`（不过它很好兼容 SQLAlchemy，Django ORM 最大的竞争者）。

因为包含 ORM，Django 能够创建功能齐全的 CRUD 应用。 CRUD（Create Read Update Delete）是 web 框架（服务器端）最美好的地方，Django 和 Flask-SQLAlchemy 使得 CRUD 操作很直接。

## Web 框架总结
写到这，web 框架出现的目的也比较明确了：隐藏基础而又烦人的处理 HTTP 请求和应答的代码。至于要隐藏多少内容，就要看框架啦。Django 和 Flask 代表了两个极端。Django 每种情况都有涉及，而 Flask 标榜自己是“微框架“，只处理 web 程序最核心的功能，依赖其他三方插件来完成其他不常用的工作。

写了这么多，记住，所有的 python web 框架功能方式都一样：它们接收 `HTTP` 请求，然后分发任务，并生成 HTML，然后返回包含 HTML 的 HTTP 应答。事实上，所有的 server 端框架（除了 Javascript 框架）都是这么工作的。希望，看完这篇文章，你已经知道 web 框架的目的，也知道怎么去选择 web 框架啦。
