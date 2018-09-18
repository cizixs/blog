---
layout: post
title: "【翻译】curl自动化http操作"
excerpt: "使用curl命令来模拟浏览器行为"
categories: 程序技术 
tags: [curl, http, linux]
comments: true
share: true
---


![curl](http://cdn.osxdaily.com/wp-content/uploads/2014/02/download-with-curl.png)

## HTTP 脚本

### 背景
这篇文档假定读者熟悉HTML和简单的网络知识。

大量应用转到网络使得HTTP 脚本被大量使用，自动化地从网络获取信息、伪装成用户、上传数据到网络服务器也变得至关重要。

`curl`是进行各种 url 操作和转换的命令行工具，这篇文章只关注`curl`在 http 请求中的应用。我想你已经知道使用`curl --help`或者`curl --manual` 来初步了解`curl`。

`curl`不能保证完成你所有的任务。它能发出请求、获取数据、发送数据和获得信息，你或许需要通过其他的脚本语言把这些结合在一起达到所需的目标。

### HTTP 协议
HTTP 是从网络服务器端获得数据的协议，是建立在`TCP/IP`协议上的简单协议。这一协议也允许把数据从客户端发送到服务器端，不过用的是不同的方法，下面我们将介绍到。

HTTP 的实质是：客户端向服务器端请求某个动作时发送`ASCII`码文件，然后服务器端返回实际的请求资源。

Curl，作为客户端就是发送 HTTP 请求的。请求由方法（GET,POST,HEAD等）、请求头部和请求的正文组成。HTTP 服务器端的响应包括：状态行（指明一切是否正常），响应头部和响应的正文。正文就是指你请求的数据，可以是html或者图像等。

### 查看协议
使用`curl`的选项`--verbose`(或者简写`-v`)可以查看`curl`发给服务器端的命令，以及其他一些信息。`--verbose`是调试和理解`curl <--> server`交互的最有用的选项。

有时候只是`--verbose`还不够，`--trace`和`--trace-ascii`会提供`curl`发送和接收的更详细信息，可以这样使用：

    curl --trace-ascii debugdump.txt http://www.example.com


### 查看时间
如果你想知道到底什么占用了那么多时间的话，或者只是想了解传输的两点间所用的时间，以及其他一些情况，你可以使用`--trace-time`选项。它将在输出中打印时间：

    curl --trace-ascii d.txt --trace-time http://www.example.com


### 查看返回内容
默认情况下，`curl`会把内容返回到`stdout`(标准输出)。你可以使用`-o`或者`-O`把它重定向到其他地方。

> 译者注： 更多关于`-o`和`-O`的详情请查看`man curl`。

## URL

### 规范
URL(Uniform Resource Locator)指定网络上每个资源的位置，你一定见过诸如`http://curl.haxx.se`或者`https://yourbank.com`的URL不下千次了。RFC 3986就是关于URL最权威的规范。

### 主机

主机名通常由DNS或者你的`/etc/hosts`文件指定的ip来解析，也就是`curl`最终通信的服务器地址。你也可以直接在url中使用ip来取代主机名。

在测试或者其他特殊情况下，你可以用`curl`的`--resolve`选项指定主机名所对应的ip地址：

    curl --resolve www.example.org:80:127.0.01 http://example.org



### 端口号
每个`curl`命令都作用在一个特定的端口上，一般是TCP端口也有可能是UDP端口。大多数情况下，你无须担心这个问题，但有时候你要在不同的端口上做测试，就需要指定端口号。端口号紧跟在主机名的冒号后，例如我们希望http协议请求`1234`端口：
    
    curl http://www.example.com:1234

服务器将使用你在URL里指定的端口号来获取服务。有时你会用到本地的代理，这个时候你就要指定代理的端口号来限定`curl`的连接服务：

    curl --proxy http://proxy.example.org:4321 http://remote.example.org


### 用户名和密码
有些需要HTTP认证的服务会要求你提供用户名和密码上传到服务器端。

你可以直接在URL里面使用用户名和密码也可以分开使用：

    curl http://user:password@example.org

或者

    curl -u user:password http://example.org

需要注意的是，这类的用户名/密码验证并不是现代用户登录的网页使用的方法，后者一般使用表单和缓存。


### 路径

URL的路径用来发送给服务器端来请求相应的资源，路径是主机名（或者端口号）后面跟的内容。

## 获取页面

### GET
HTTP 最常见也最简单的操作就是获取一个 URL 的内容，URL 指向的可以是网页、图像或者文件。客户端向服务器端发起 GET 请求，然后收到想要的文档。如果你使用下面的命令：

    curl http://curl.haxx.se

就会在终端得到URL所指向网页的所有内容。

HTTP响应会包含头部信息，而默认情况下它是隐藏的，可以使用 `curl`的`--include(-i)`选项来显示头部。

### HEAD
你也可以使用`--head（-I<大写的i>）`选项**只请求**头部信息，这时候`curl`会发送一个HEAD请求。在某些特殊的情况下，服务器端会拒绝HEAD请求，而其他的请求都是正常的。

HEAD请求就是告诉服务端只返回和GET请求完全一样的头部，不返回响应正文。也就是说你可以看到头部的`Content-Length`标签，但是没有具体的正文。

## HTML表单

### 表单的解释
表单用来在 HTML 显示可供用户输入的域，通过点击 *OK* 或者 *submit* 把输入的数据发送给服务器。服务器端根据得到的数据来决定接下来的行为。比如，用输入的词来进行搜索，或者把信息保存到缺陷跟踪系统，在地图上显示输入的地址，或者用这些信息验证用户登录以确定他是否可以看到更多的内容。

当然，服务器端必须要有响应的程序来处理输入的数据，毕竟这些东西不可能无中生有。

### GET
`GET-form`使用 GET 方法，在 HTML 中是这样的：


     <form method="GET" action="junk.cgi">
     <input type=text name="birthyear">
     <input type=submit name=press value="OK">
     </form>


在你最喜欢的浏览器里，你会看到这个表单包括一个文本框和一个`OK`按钮。如果你填入`1905`，然后点击按钮，浏览器就会生产一个url。这个url会把`junk.cgi?birthyear=1905&press=OK`附加到前面url（表单所在的url）的路径部分。

如果原来的表单出现在`www.hotmail.com/when/birth.html`，那么你得到的第二个页面将是`www.hotmail.com/when/junk.cgi?birthyear=1905&press=OK`。

大多数的搜索引擎也是这么工作的。

要使用`curl`来完成 GET 表单的功能，直接输入预期的 url 就行：

    curl www.hotmail.com/when/junk.cgi?birthyear=1905&press=OK


### POST
GET 方法把所有的输入项都放在浏览器的 url 上。如果你想把该页保存为书签，这当然是个不错的方法。这种方法的缺点也是显而易见的：当你输入的是保密信息或者有很长输入域的时候也会产生特别长的 url。

HTTP 协议提供了另外一种方法：POST。 用户发送的信息和 url 是独立的，所以你就不会在 url 里看到输入的数据。

这时候的表单和前面的大体一样：

     <form method="POST" action="junk.cgi">
     <input type=text name="birthyear">
     <input type=submit name=press value=" OK ">
     </form>

要使用`curl`来实现这一方法，我们可以使用：

    curl --data "birthday=1905&press=%20OK%20" http://www.example.com/when.cgi

这种 POST 会使用 `Content-Type： application/x-www-form-urlencoded`头部选项，也是最普遍使用的 POST 方法。

你发送到服务器的数据必须是提前编码好的，`curl`不会自动给你编码。如果你想数据里包含空格，你必须要用`%20`来替换。不进行编码会导致收到的数据是错误的。

实际上，当前的`curl`版本已经支持自动 url 编码了：

    curl --data-urlencode "name=I am Dnaniel" http://www.example.com


### POST上传文件
在1995年的时候，就已经定义了HTTP POST数据的另一种方式，它出现在RFC 1867文档里，因此也被称为`RFC1867-posting`。

这个方法主要是为了更好地支持文件上传设计的。可以让用户上传文件的表单在 HTML 里可以写成这样：
    
    <form method="POST" enctype='multipart/form-data' action="upload.cgi">
     <input type=file name=upload>
     <input type=submit name=press value="OK">
    </form>


这表明要上传的数据格式为`multipart/form-data`， 要使用`curl`来实现这样的表单上传，可以使用下面的命令：
```
curl --form upload=@localfilename --form press=OK [URL]
```

### 隐藏域

HTML 应用在页面之间传递状态信息的一个常用方式是在表单中使用隐藏域。隐藏域是默认填写好的，但不会展示给用户，和其他域一样被传递。

包含一个可见域、一个隐藏域还有一个提交按钮的简单表单如下所示：
    
    <form method="POST" action="foobar.cgi">
     <input type=text name="birthyear">
     <input type=hidden name="person" value="daniel">
     <input type=submit name="press" value="OK">
    </form>


要用`curl`来实现隐藏域并不需要考虑额外的东西，直接把它当做正常的域就行：

    curl --data "birthyear=1905&press=OK&person=daniel" [URL]


### 弄清POST的样子
当你使用`curl`来填充表单发送到服务器的时候，自然会想弄明白浏览器发送 POST 请求的方式。

有一个简单的方法：把 HTML 页面存储下来，把 form 的`method`改成 GET，点击提交按钮。你就可以在 URL 栏看到浏览器发送的数据。

## HTTP 上传

### PUT
上传数据到HTTP服务器的最佳方式是 PUT，再次提醒，服务器端必须有相应的程序来处理接收的 HTTP 数据流。

把文件上传到服务器的命令：

    curl --upload-file uploadfile http://www.example.com/reveive.cgi


## HTTP验证

### 基本验证
HTTP 验证是告诉服务端你的用户名和密码，这样服务端来判断你是否有权限访问特定的资源。基本的验证是基于明文的，也就是说用户名和密码的发送只是稍微不那么明显，但是在你和服务端的嗅探器是可读到的。

`curl`使用用户名和密码的验证如下：

    curl --user name:password http://www.example.com


### 其他验证
网站可能会需要其他的验证方式（查看服务端返回的头部），那么或许 `--ntlm`,` --digest`, `--negotiate` 或者 `--anyauth`会适合你。

### 代理验证
有时候，你的 HTTP 访问只能通过特定的 HTTP 代理才能完成，这在公司比较普遍。HTTP 代理器需要自己的用户名和密码来允许客户端访问互联网。使用`curl`来指定这些的话，就要这样：

    curl --proxy-user proxyuser:proxypass curl.haxx.se


如果你的代理需要 NTLM 方法认证，就用 `--proxy-ntlm`，如果需要 Digest,就用` --proxy-digest`。

如果你用上面方式的时候，没有指定密码，`curl`会提示你输入密码。

### 隐藏的凭据
要注意，程序启动的时候，可以通过列出系统的进程来查看它的参数。如果你使用明文密码的话，其他用户就可以看到。

需要注意的是，尽管 HTTP 的验证是这么工作的，很多的网站不会采用这种登录方式进行验证。你可以阅读下面的`WEB登录`章节来了解更多细节。

## 更多HTTP 头部选项

### Referer
HTTP 请求可以包含`referer`域，来指明客户端通过什么 URL 来访问该资源的。有些程序通过查看这一选项来核实请求不是来自外部或者未知的地方。虽然这样根据可任意伪造的项来判断是很蠢的做法，很多程序还是会这么做。使用`curl`，你可以把任意内容放到 referer 域，以此来欺骗服务器来响应你的请求.

     curl --referer http://www.example.come http://www.example.com


### User Agent
和 referer 域类似，所有的 HTTP 请求也可以设置 `User-Agent`域,它指明用户使用的客户端。许多程序根据该选项决定如何渲染页面。愚蠢的程序员会对不同的浏览器编写不同的页面，来让所有的页面在不同的浏览器上达到最佳效果。当然也包括重写javascript、vbscript等。

有时候，你发现通过`curl`获得的页面和浏览器获得的页面是不一样的。那你就需要修改`user-agent`选项来欺骗服务器，让它以为你使用的是某个浏览器。

要让`curl`变成 windows 2000 的 IE5，使用：

    curl --user-agent "Mozilla/4.0 (compatible; MSIE 5.01; Windows NT 5.0)" [URL]


要让`curl`变成 Linux 下的 Netscape4.73 ， 使用：

    curl --user-agent "Mozilla/4.73 [en] (X11; U; Linux 2.2.15 i686)" [URL]


## 重定向

### 位置头部
当某资源从服务器上被获取时，服务器的回应可以包含浏览器下一步去哪里找到该资源的提示。头部中提示浏览器重定向的选项就是`Location`。

`curl`默认情况下并不追踪头部的位置选项，而是直接把重定向回应像其他应答一样显示出来。当然`curl`有选项可以来追踪头部的重定向，那就是`--location`选项:

    curl --location http://www.example.com

如果你使用`curl`来 POST 请求到某网站，而该网站立即把你重定向到另外的页面，你可以结合`--location(-L)`和`--data/--form`一起使用。Curl只会在第一个请求中使用 POST，而在后面的请求使用GET方法。


### 其他重定向
浏览器一般至少支持两种`curl`无法支持的重定向：一种是HTTP头部包含某个标签，让浏览器在特定秒之后重新获取特定的 URL，还有就是使用 javascript 来获取新的 URL。

## Cookie

### cookie基础
网页浏览器使用Cookie来保持客户端的状态。Cookie只是一系列被赋值的名字而已，通过服务器被发送到客户端。服务器端会告诉客户端它希望该cookie被发送回来的主机名和地址，也包括过期日期和其他的一些属性。

很多服务器和程序使用cookie把一系列的请求归类到一个会话（session）。为了保持这一状态，我们必须能够保存cookie，并通过服务希望的方式把cookie发送回去。这也是浏览器处理cookie的方式。

### cookie选项

把 cookie 发送到服务器端最简单的方式是直接把它们添加到命令行里：

    curl --cookie "name=Daniel" http://www.example.com


cookie 通过正常的 HTTP 请求在头部被发送出去。要用`curl`来记录 cookie 可以使用`--dump-header(-D)`选项：

    curl --dump-header headers_and_cookies http://www.example.com

(下面介绍的`--cookie-jar`是使用 cookie 更好的方式。)


`curl`内部提供了 cookie 解析的引擎，可以让你方便地连接到服务器端，并发送前面的连接已经保存的 cookie（或者手动伪造的 cookie 让服务器端相信你有过上一个连接）。要使用以前存储的 cookie，你可以这样：

    curl --cookies stored_cookies_in_file http://www.example.com


当你使用`--cookie`选项的时候，`curl`的 cookie 引擎就会自动开启。如果你只想让`curl`理解要接收的 cookie，你可以指定一个不存在的文件。比如，如果你想让`curl`理解某个页面的 cookie，并追踪重定向的话，可以这么用：

    curl --cookie nada --location http://www.example.com


`curl`能和 Netscape 与 Mozilla 使用相同的文件格式来处理 cookie，在脚本间传递 cookie 也就很方便。`--cookie(-b)`选项自动检测给定参数是否为 cookie 文件并解析其中的 cookie，而`--cookie-jar(-c)`选项，你可以让`curl`把新的 cookie写入到文件里：


    curl --cookie cookies.txt --cookie-jar newcookies.txt  http://www.example.com


## HTTPS

### HTTPS 是安全的 HTTP
安全 HTPP 协议的传输有很多种方法，现在最常用的协议是 HTTPS，就是基于 SSL 的 HTTP 协议。SSL 把要传输和接收的数据加密，使得对敏感信息的攻击变得困难。
SSL 这种加密传输和密钥管理机制有很多的优势。

归功于开源的 OpenSSL 库，`curl`也支持加密的操作。从 HTTPS 服务器获得页面的只需要简单运行`curl`：

    curl https://secure.example.com


### 证书
在 HTTPS 的世界里，证书作为密码的补充来证明你的身份。`curl`支持客户端的证书，所有的证书都要求你输入密码来使用。这个密码可以直接在命令行指定，也可以在`curl`提示时交互式输入。使用证书来获取 HTTPS 的内容：

    curl --cert mycert.pem https://secure.example.com

`curl`也会把服务器端的证书与本地存储的 CA 证书进行验证，验证失败会导致`curl`拒绝连接。你也可以使用 `--insecure(-k)`选项来告诉`curl`忽略验证失败的服务器。

更多关于服务器证书和ca的问题可以访问下面的[文档](http://curl.haxx.se/docs/sslcerts.html)。

## 自定义请求元素

### 修改方法和头部
要完成一些很 cool 的东西，你可能需要增加或者修改`curl`请求的头部元素。

例如，你可以把 POST 方法改为 PROPFIND，发送类型为`Content-Type: text/xml`的数据：

    curl --data "<xml>" --header "Content-Type: text/xml" --request PROPFIND url.com

通过传递一个没有值的头部，你可以删除原来的项。例如，你可以这样删除 Host: header：

    curl --header "Host:" http://www.example.com


这种方式也可以用来添加头部的选项。如果服务器需要`Destination`的头部，你可以加上去：

    curl --header "Destination: http://nowhere" http://example.com


### 修改方法（续）
需要注意的是，`curl`会根据自己被传递的方法来进行 HTTP 的请求，比如`-d`表示 POST，`-l`表示 HEAD。如果你是用`curl`的`--request/-X`选项,`curl`会改变它方法的关键字，但是行为却不会改变。也就是说，如果你用`-d` data 来做 POST 请求，然后用`-X`来修改方法为 PROPFIND，`curl`仍然会发送 POST 请求。你可以把 GET 请求这么改成 POST 请求：

    curl -X POST http://example.com

但是`curl`还是会执行 GET 的动作，不会发送任何的正文。

## 网页登陆

### 登录技巧
虽然登陆不属于 HTTP 的内容，但是会给很多学习 HTTP 的人造成困惑，所以这里就说说大部分的登录框是怎么工作的，以及怎么使用`curl`来完成登录的工作。

需要注意的是，虽然可以自动化地执行登录动作，你还是需要些不少脚本来完成`curl`的登录命令的。

首先，服务器端大多用 cookies 来追踪客户端的登录信息的，所以你必须要获取响应里的 cookies。其次，很多网站在登陆页面也会设置特殊的 cookie（以此确定你是从登陆页面进来的），你也必须先到登陆页面来获取响应的 cookies。

不少网页的登录系统使用大量的 javascript 来设置和修改 cookie 的内容，可能是为了防止程序化的自动登录，比如这篇文章讲到的... 如果阅读源码不能让你清晰地手动重复登录的过程，抓取浏览器发送和接收的 HTTP 包来分析其中的 cookies 是一定可行的捷径。

在实际的`<form>`标签里，会有很多填充的随机内容或者秘密生成的隐藏域，你要先捕获这些 HTML 代码，找到所有登录需要的内容，添加到 POST 的请求里。记住在正常 POST 之前，这些数据必须进行 URL 编码。

## 调试

### 一些调试技巧
有些时候，你用`curl`命令获取的网页信息和你用浏览器的不一样，那你就要把`curl`的请求尽量变得和浏览器的一致。

+ 使用`--trace-ascii`选项来存储请求所有的详细日志，以便后续的理解和分析
+ 确保你在必要时使用了 cookie（包括使用`--cookie`读取 cookie和`--cookie-jar`写 cookie）
+ 和浏览器设置相同的`user-agent`选项
+ 和浏览器设置相同的`referer`选项
+ 如果你使用 POST，确保发送的数据包括所有的域，并且顺序和浏览器保持一致

firefox 下的 LiveHTTPHeader 插件可以帮你查看发送和接收请求的头部信息，chrome 的开发者工具也能完成相同的功能。

还可以使用 ethereal 或者 tcpdump 工具捕获网络上的 HTTP 流量来检查浏览器发送和接收的 HTTP 头部。（HTTPS 让这一方法很没有效率可言）

## 参考

### 协议标准

+ RFC 2616：如果你想深入理解 HTTP 协议，就必须阅读该标准。
+ RFC 3986：解释 URL 的语法
+ RFC 1867：定义了 HTTP 上传文件的格式
+ RFC 6525：定义了 HTTP cookies 的工作方式

### 网站

+ [cURL的主页](http://curl.haxx.se)

[原文链接](http://curl.haxx.se/docs/httpscripting.html)

> Written with [StackEdit](https://stackedit.io/).
