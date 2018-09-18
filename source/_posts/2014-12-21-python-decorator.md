---
layout: post
title: "解析 python decorator "
excerpt: "学会 decorator 是理解 python 的第一个里程碑。"
categories: 程序技术
tags: [python, decorator]
comments: true
share: true
---

首次看到 python 的 decorator 的时候，感到非常新奇，也非常困惑。这个看起来很酷，在各种源码里经常出现的家伙到底是什么意思，怎么使用？

    @myDecorator
    def aFunction():
        print("inside a Function")

这篇文章就解释一下自己的这个疑惑。

## 从函数开始

函数可以看做一个容器，它根据接受的参数进行运算，并返回出结果（没有返回的函数可以看做返回值是 `None` ）。

![](http://t1.qpic.cn/mblogpic/a51abdb4aab5f9027162/460.jpg)

在 Python 这样的高级语言里面，函数还有另外一个身份：对象。这就说明函数不仅是动词（对值做运算的过程），也是名词（本身是可以使用的对象）。这样，函数就能像 `integers`, `strings`, `objects` 一样被传来传去。

想到这里的话，那可以把一个函数作为参数传给另外一个函数，或者函数的返回值也是一个函数啊！这就是理解 decorator 的基础和最关键的一点啦！

Decorator 就是这样一个函数：它接受一个函数作为参数，并返回一个函数作为返回值。

> A decorator is a function that takes a function object as an argument, and returns a function object as a return value.


## 走向 decorator

下面是一个简单装饰器的定义：

    def verbose(func):
    
        def new_function(*args, **kwargs):
            print("Entering")
            func(*args, **kwargs)
            print("Exiting ")
    
        return new_function

它在执行原来的函数前和执行后，各打印一句话。那怎么使用这个装饰器呢？很简单，根据前面的定义，传给它一个函数就行。

假如我们有一个 `hello world` 的函数：

    def hello_world():
        print("Hello,world!")

那么执行下面的语句

    v_hello_world = verbose(hello_world)

我们就得到一个 v_hello_world 的变量，根据前面的定义，这个变量是一个函数。**需要注意的是，函数只有在遇到 `()` 符号的时候才会调用**。我们新得到的函数还没有被调用，也就是说接下来可以调用它，让它执行定义的内容。

    v_hello_world()

这个时候会看到下面的输出：

    Entering hello_world
    hello,world!
    Exiting hello_world

其实可以不用引入新的变量，直接这样写：

    hello_world = verbose(hello_world)

这里 `verbose` 被称为装饰函数，它用来给原来的函数添加新的特性。

## @ 符号
为了更好地使用上述的特性，`Python` 提供了 `@` 这个[语法糖](http://en.wikipedia.org/wiki/Syntactic_sugar)。这样的话，上面的例子就可以简写为：

    @verbose
    def hello_world():
        print("Hello,world!")

得到的就是被装饰后的函数。在刚开始不熟悉的时候，可以把装饰器还原成 `hello_world = verbose(hello_world)` 的形式，方便自己的理解。


## decorator 高级用法

### 嵌套（chained）的 decorator
事实上，decorator 是可以嵌套使用的，下面的代码来自 stackoverflow 的[这个答案](http://stackoverflow.com/questions/739654/how-can-i-make-a-chain-of-function-decorators-in-python/739665#739665)。

    def makebold(fn):
        def wrapped():
            return "<b>" + fn() + "</b>"
        return wrapped
    
    def makeitalic(fn):
        def wrapped():
            return "<i>" + fn() + "</i>"
        return wrapped
    
    @makebold
    @makeitalic
    def hello():
        return "hello world"
    
    print hello() ## returns <b><i>hello world</i></b>

### 带参数的 decorator
除了可以嵌套使用之外，decorator 还能够带有参数。为了说明这个问题，假设有这样的场景：我们需要一个 decorator 把原来的函数执行多次。

如果要执行的次数是固定的，比如 3 次，问题还是很容易解决：

    def repeat3(func):
        '''
        execute original function three times
        '''
        def inner(*args, **kwargs):
            func(*args, **kwargs)
            func(*args, **kwargs)
            func(*args, **kwargs)
    
        return inner

如果要执行的次数不确定呢？我们就需要一个额外的参数，最终的目的就是下面的代码能够把 `hello_world` 函数执行 3 遍：

    @repeat(3)
    hello_world():
        print("Hello,world!")


根据前面的定义，上面的代码等价于：

    hello_world = repeat(3)(hello_world)

那么 `repeat(3)` 的结果必须是一个函数，才能在后面被传参调用。不仅如此， `repeat(3)` 的结果接受的参数还是函数，那它应该也是 decorator。也就是说，`repeat` 是一个函数，它的返回值也是函数，并且返回的函数的参数和返回值都是函数。好吧，我也是晕了！还是看个例子吧：

    def repeat(n):
        def repeatn(f):
            def inner(*args, **kwargs):
                for i in range(n):
                    f(*args, **kwargs)
            return inner
        return repeatn
    
    @repeat(5)
    def hello(name="world"):
        print("Hello,%s" % name)
    
    hello()

再解释一遍，`repeat` 接受 5 作为参数，并返回一个 decorator —— `repeatn`，`repeatn` 返回一个把原来的函数执行 n 遍的函数。

### 他们真的一样吗？

写到这里，我们已经知道：装饰器就是把原来的函数记作 f1 添加一些特性，然后生成一个新的函数 f2。然后，我们就可以想使用原函数 f1 那样使用 f2，不用在意内部的细节。但是这两个函数真的是一样的吗？**答案是否定的**。我们来看一下例子：

定义一个简单的函数，

    >>> def bar():
    ...   ''' This is bar function document '''
    ...   pass

查看 bar 函数的属性，

    >>> bar.__name__, bar.__doc__, bar.__module__
    ('bar', ' This is bar function document', '__main__')

使用 inspect 查看 bar 函数的参数定义，

    >>> import inspect
    >>> inspect.getargspec(bar)
    ([], None, None, None)

    >>> bar2=verbose(bar)
    >>> bar2.__name__, bar2.__doc__, bar2.__module__
    ('shown', None, '__main__')

    >>> inspect.getargspec(bar2)
    ([], 'args', 'kwargs', None)

好了，不难理解，原函数 f1 的属性并没有传递到新的函数 f2。那么 f1 的属性就不能使用了，而我们对 f2 的属性根本就不感兴趣。那么直接一点的方法就是把 f1 的属性拷贝到 f2：

    def verbose(func):
        def new_function(*args, **kwargs):
            print("Entering")
            func(*args, **kwargs)
            print("Exiting ")
    
    
        new_function.__name__ = func.__name__
        new_function.__doc__ = func.__doc__
        new_function.__module__ = func.__module__
        new_function.__dict__.update(func.__dict__)
    
        return new_function
    
    
    >>> bar2=verbose(bar)
    >>> bar2.__name__, bar2.__doc__, bar2.__module__
    ('bar', ' This is bar function document ', '__main__')

Python2.5 以后的版本在 functools 提供了一个装饰器(没错，谁的问题谁负责)来解决 `wraps` 这个问题：

    >>> from functools import wraps
    
    >>> def verbose(f):
    ...   @wraps(f)
    ...   def new_function(*args, **kwargs):
    ...     print("Entering")
    ...     func(*args, **kwargs)
    ...     print("Exiting ")
    ...   return new_function
    
    >>> bar2=verbose(bar)
    >>> bar2.__name__, bar2.__doc__, bar2.__module__
    ('bar', ' This is bar function document', '__main__')
    
### 函数并不是唯一
我们上面的文章一直在讲用函数实现的 decorator，其实函数并不是唯一的实现方式，利用类也能实现 decorator。让我们再次回到 decorator 的本质：

    my_func = decorator(my_func)

看到了没？decorator 是这样的东西（不一定是函数）：它**可以被调用**，接受一个函数作为参数，调用后会返回一个新的函数。你可能会问，这个东西除了函数还会是什么？答案就在下面两点：

1. 函数也是类，因为在 python 里一切都是对象
2. 一个东西能被调用，是因为内部定义了 `__call__` 函数。如果我们自己写的类实现了这个特殊的函数，也能够被调用啦。

好了，知道了原理，我们就来看看用类实现的 decorator，下面的代码来自于[这篇文章](http://www.artima.com/weblogs/viewpost.jsp?thread=240808)：

    class entryExit(object):
        def __init__(self, f):
            self.f = f

        def __call__(self):
            print "Entering", self.f.__name__
            self.f()
            print "Exited", self.f.__name__


    class decoratorWithArguments(object):
        def __init__(self, arg1, arg2, arg3):
            """
            If there are decorator arguments, the function
            to be decorated is not passed to the constructor!
            """
            print "Inside __init__()"
            self.arg1 = arg1
            self.arg2 = arg2
            self.arg3 = arg3

        def __call__(self, f):
            """
            If there are decorator arguments, __call__() is only called
            once, as part of the decoration process! You can only give
            it a single argument, which is the function object.
            """
            print "Inside __call__()"
            def wrapped_f(*args):
                print "Inside wrapped_f()"
                print "Decorator arguments:", self.arg1, self.arg2, self.arg3
                f(*args)
                print "After f(*args)"
            return wrapped_f

    @decoratorWithArguments("hello", "world", 42)
    def sayHello(a1, a2, a3, a4):
        print 'sayHello arguments:', a1, a2, a3, a4

    print "After decoration"

    print "Preparing to call sayHello()"
    sayHello("say", "hello", "argument", "list")
    print "after first sayHello() call"
    sayHello("a", "different", "set of", "arguments")
    print "after second sayHello() call"

## decorator 实际中的使用
花费了这么多的精力和脑细胞学习这个很炫的东西，那么 decorator 在实际的编码中到底有哪些用呢？这个嘛，用处可多了！

比如，稍微修改一下上面例子的代码，就能写一个函数的计时器：

    def timer(func):
        """ A decorator to print the time function executes """
        import time
        def wrapper(*args, **kwargs):
            start = time.time()
            func(*args, **kwargs)
            elapse = (time.time() - start) * 1000
            print "func %s elapsed time: %f ms" % (func.__name__, elapse)

    @timer
    foo():
        do_something()

又比如，上面提过的 [stackoverflow 的帖子](http://stackoverflow.com/a/1594484/1925083)也给出了类似的 benchmark，counter，logging 装饰器。



    def logging(func):
        """
        A decorator that logs the activity of the script.
        (it actually just prints it, but it could be logging!)
        """
        def wrapper(*args, **kwargs):
            res = func(*args, **kwargs)
            print func.__name__, args, kwargs
            return res
        return wrapper


    def counter(func):
        """
        A decorator that counts and prints the number of times a function has been executed
        """
        def wrapper(*args, **kwargs):
            wrapper.count = wrapper.count + 1
            res = func(*args, **kwargs)
            print "{0} has been used: {1}x".format(func.__name__, wrapper.count)
            return res
        wrapper.count = 0
        return wrapper




Python 自带的 decorator 就有 `staticmethod` 和 `property`。除此之外，你还可以在[这个 wiki ](https://wiki.python.org/moin/PythonDecoratorLibrary)上面查看更多的实例。

最后，更多的实例就要等你在实际中开发啦！

## 参考资料

- [Python Decorators](http://pythonconquerstheuniverse.wordpress.com/2012/04/29/python-decorators/)
- [Understanding Python Decorators in 12 Easy Steps!](http://simeonfranklin.com/blog/2012/jul/1/python-decorators-in-12-steps/)
- [How can I make a chain of function decorators in Python?](http://stackoverflow.com/questions/739654/how-can-i-make-a-chain-of-function-decorators-in-python/1594484#1594484)
- [Python Decorator Library](https://wiki.python.org/moin/PythonDecoratorLibrary)
- [Decorators by Kent S Johnson](http://kentsjohnson.com/kk/00001.html)
- [functools.wraps](https://docs.python.org/2/library/functools.html#functools.wraps)
- [A guide to Python's function decorators](http://thecodeship.com/patterns/guide-to-python-function-decorators/)
