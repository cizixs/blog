---
layout: post
title: "bottle 源码解析"
excerpt: "bottle 是一个极简的 python web 框架，这篇文章会对它的源码进行简单的分析。"
categories: blog
tags: [python, bottle, web, framework, wsgi, http]
comments: true
share: true
---

这篇文章不会讲如何使用 bottle，如果感兴趣，直接看它们的官方文档。
主要讲讲 web 框架的一些东西，阅读之前最好对下面的这些知识有一定的了解：

+ 熟悉 HTTP 协议
+ 对 python 有一定的了解
+ 知道 wsgi 是干什么的
+ 写过 web 应用

如果不太了解 web 框架，可以阅读我之前的一篇文章：[什么是 web 框架？](http://cizixs.com/2015/09/21/what-is-a-web-framework/)。

## 简介
bottle 是一个极简的 python web 框架，可以用来用来快速搭建 web 应用，并不是开发复杂项目的第一选择，因为它并没有提供配置文件集成，数据库管理，可扩展的中间件等特性。

bottle 只是实现了 web 框架最核心的部分，因此代码很少。事实上，这篇文章参考的 bottle 0.12.9 版本只有 4k 行代码，如果去除很长的 import 头部、文档、三方 wsgi server 的支持，最核心的代码估计只有 2k-3k 行，对于我们学习 web 框架提供了便利。

在阅读 bottle 代码的时候，要学会思考：什么是 web 框架最核心的功能？为什么要这么写？有哪些功能实现的不好？怎该怎么修改？

就我个人而言，觉得 bottle 在下面几个方面表现得并不好：

+ 代码风格没有特定的规范：虽然都是空格、某行太长、缩进不规范，一行有多个表达式这种小问题，但对于开源项目来说，遵守 code style 是很重要的。好在 bottle 很小，不会因为这个带来大的问题
+ python2 和 python3 的兼容处理的不好，在文件的头部有一堆 import。当然你可以说这是 python 的问题！
+ 所有的文件都在同一个文件，把文件分成不同的模块会更好理解。当然我理解这么做事为了突出 bottle 的小：只要一个文件就搞定！
+ request 和 response 都是全局变量，所有要使用到请求 context 的时候都要访问全局变量，这种设计是 anti-pattern 的。

## hello world

我们来看看 bottle 的 hello world 怎么写：

    from bottle import route, run, template

    @route('/hello/<name>')
    def index(name):
        return template('<b>Hello {{name}}</b>!', name=name)

    run(host='localhost', port=8080)

后面所有的分析都会基于这个 hello world，所以最好自己创建个文件运行 一下，感受感受。

使用浏览器打开或者使用 curl：

    curl http://localhost:8080/hello/world

看看结果。

## 分析

好了，我们看看上面的 hello world 程序有哪几部分组成：

+ 路由（route）：这是 web 框架最关键的功能，把请求转发到对应的处理函数
+ 运行（run）：提供 wsgi server，能够监听所有请求并传递给后面的应用（就是 bottle 应用）
+ 模板（template）：动态生成 HTML 文件，发送给客户端

除了上面的三个模块，还有两个隐藏的东西没有在代码中出现：

+ 请求（request）：对 HTTP 请求的封装
+ 响应（response)：对 HTTP 响应的封装

下面会逐个分析这五个部分的内容，在那之前我们先看看整个流程的流转过程：

1. wsgi 启动，加载 bottle 应用，监听在设置的端口
2. 请求进来之后，被 wsgi server 封装好，调用 Bottle 的 `__call__` 函数，交给 bottle 处理
3. bottle 根据传过来的 environ 字典，初始化 request 和 response 对象
4. bottle 获取传过来的 url 值，匹配之前根据代码中装饰器生成的路由器对象，找到处理函数
5. 调用处理函数
6. 处理函数会使用默认的模板引擎，替换所有的变量，返回处理结果
7. 把处理函数返回的值封装成 wsgi 兼容的对象
8. 把封装好的 response 返回给 wsgi server

每个 bottle 应用都是一个 bottle app，上面代码中没有显示定义，默认是

    app = default_app = AppStack()
    app.push()

而 AppStack 的定义是：

    class AppStack(list):
        """ A stack-like list. Calling it returns the head of the stack. """

        def __call__(self):
            """ Return the current default application. """
            return self[-1]

        def push(self, value=None):
            """ Add a new :class:`Bottle` instance to the stack """
            if not isinstance(value, Bottle):
                value = Bottle()
            self.append(value)
            return value

### 路由

bottle 的路由一般使用 python 的装饰器实现，比如上面的 `@route('/hello/<name>')`，route 装饰器的实现是这样的：

    def route(self,
              path=None,
              method='GET',
              callback=None,
              name=None,
              apply=None,
              skip=None, **config):

        if callable(path): path, callback = None, path
        plugins = makelist(apply)
        skiplist = makelist(skip)

        def decorator(callback):
            if isinstance(callback, basestring): callback = load(callback)
            for rule in makelist(path) or yieldroutes(callback):
                for verb in makelist(method):
                    verb = verb.upper()
                    route = Route(self, rule, verb, callback,
                                  name=name,
                                  plugins=plugins,
                                  skiplist=skiplist, **config)
                    self.add_route(route)
            return callback

        return decorator(callback) if callback else decorator

做的事情就是解析参数和 callback 函数，生成多个 route 对象，添加到自己的列表中，以备后续的查询和使用。在我们的例子中，rule 就是 `'/hello/<name>'`，verb 就是 `GET`，callback 就是定义的函数 `index`。

add_route 方法实现如下：

    def add_route(self, route):
        """ Add a route object, but do not change the :data:`Route.app`
            attribute."""
        self.routes.append(route)
        self.router.add(route.rule, route.method, route, name=route.name)
        if DEBUG: route.prepare()

主要做了两件事：把生成的 route 对象加入到内部的变量列表中，和调用 router 的 add 方法。后面这件事情比较复杂，但最核心的逻辑是：如果是静态路径，直接添加到对应的变量中；如果是动态路径，还要经过正则匹配和编译，再把结果添加到对应的变量。

而具体的调用处理函数的逻辑在 `Bottle._handle(self, environ)`，比较核心的部分如下：

    def _inner_handle():
            # Maybe pass variables as locals for better performance?
            try:
                route, args = self.router.match(environ)
                environ['route.handle'] = route
                environ['bottle.route'] = route
                environ['route.url_args'] = args
                return route.call(**args)
            except HTTPResponse:
                return _e()
            except RouteReset:
                route.reset()
                return _inner_handle()
            except (KeyboardInterrupt, SystemExit, MemoryError):
                raise
            except Exception:
                if not self.catchall: raise
                stacktrace = format_exc()
                environ['wsgi.errors'].write(stacktrace)
                return HTTPError(500, "Internal Server Error", _e(), stacktrace)

调用 `router.match` 进行匹配，返回对应的 route 对象和参数，然后调用 `route.call(**args)`来执行业务逻辑。

### 请求
请求对象比较简单，就是对 environ 变量的包装，添加了一下 bottle 相关的内容，方便访问某些信息。

初始化传入的参数就是 `environ`:

    def __init__(self, environ=None):
        """ Wrap a WSGI environ dictionary. """
        #: The wrapped WSGI environ dictionary. This is the only real attribute.
        #: All other attributes actually are read-only properties.
        self.environ = {} if environ is None else environ
        self.environ['bottle.request'] = self

值得一提的是，为了方便访问某些信息，比如请求的头部、cookie 等，request 对象把它们封装成 `DictProperty`：

    @DictProperty('environ', 'bottle.request.headers', read_only=True)
    def headers(self):
        """ A :class:`WSGIHeaderDict` that provides case-insensitive access to
            HTTP request headers. """
        return WSGIHeaderDict(self.environ)

这个 `DictProperty` 是一个描述器（descriptor）：

    class DictProperty(object):
        """ Property that maps to a key in a local dict-like attribute. """

        def __init__(self, attr, key=None, read_only=False):
            self.attr, self.key, self.read_only = attr, key, read_only

        def __call__(self, func):
            functools.update_wrapper(self, func, updated=[])
            self.getter, self.key = func, self.key or func.__name__
            return self

        def __get__(self, obj, cls):
            if obj is None: return self
            key, storage = self.key, getattr(obj, self.attr)
            if key not in storage: storage[key] = self.getter(obj)
            return storage[key]

        def __set__(self, obj, value):
            if self.read_only: raise AttributeError("Read-Only property.")
            getattr(obj, self.attr)[self.key] = value

        def __delete__(self, obj):
            if self.read_only: raise AttributeError("Read-Only property.")
            del getattr(obj, self.attr)[self.key]

只有第一次访问的时候会调用函数生成，因为它会把生产的值添加到对象的变量 environ 中，后面的调用会直接用之前生成的值。

### 响应（response）
你也许已经猜到了，Response 对象就是对 http 响应的以下部分的封装：

+ status line
+ headers
+ body

这部分的内容也不多讲了，如果想了解直接看代码，也比较简单。

### wsgi server

当所有的代码都准备好了之后，剩下的就是运行了。因为 [PEP 0333-WSGI](https://www.python.org/dev/peps/pep-0333/) 协议规范，已经有很多优秀的 WSGI server 可以直接使用，所以 bottle 里面这部分也是直接使用外部的 WSGI server 来运行的，默认使用的是 `wsgiref.simpe_server`，不过支持的 server 很多：

+ cgi
+ cherrypy
+ paste
+ tornado
+ twisted
+ gunicorn
+ eventlet
+ gevent
+ ……


### 模板
目前支持三种模板引擎：Mako，Cheetah，和 Jinja2。使用起来也很简单，把相关的参数组织以下，调用模板引擎的对应函数就行，Jinja2 为例：

    class Jinja2Template(BaseTemplate):
        def prepare(self, filters=None, tests=None, globals={}, **kwargs):
            from jinja2 import Environment, FunctionLoader
            self.env = Environment(loader=FunctionLoader(self.loader), **kwargs)
            if filters: self.env.filters.update(filters)
            if tests: self.env.tests.update(tests)
            if globals: self.env.globals.update(globals)
            if self.source:
                self.tpl = self.env.from_string(self.source)
            else:
                self.tpl = self.env.get_template(self.filename)

        def render(self, *args, **kwargs):
            for dictarg in args:
                kwargs.update(dictarg)
            _defaults = self.defaults.copy()
            _defaults.update(kwargs)
            return self.tpl.render(**_defaults)

        def loader(self, name):
            if name == self.filename:
                fname = name
            else:
                fname = self.search(name, self.lookup)
            if not fname: return
            with open(fname, "rb") as f:
                return f.read().decode(self.encoding)

### 其他

除了上面主要的功能之外，bottle 还有一些特性：

+ 支持 hook，能够在处理请求之前和之后执行用户添加的 hooks
+ 可以引入三方的 middleware，比如使用 Beaker 来添加 session 的功能
+ app mount：能够添加 wsgi 兼容的 app 到某个 url 下，这样就可以和其他框架一起工作，扩展 bottle 的功能
+ plugins：支持插件，帮助完成一些重复的工作，或者集成三方的功能

## 参考

+ [bottle 官方文档](http://bottlepy.org/docs/dev/index.html)
+ [bottle github 上的源码](https://github.com/bottlepy/bottle)
