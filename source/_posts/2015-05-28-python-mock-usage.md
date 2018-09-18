---

layout: post
title: "python mock 库的使用"
excerpt: "在 unittest 中模拟掉外部的服务"
categories: blog
tags: [unitest，mock, python]
comments: true
share: true
---

![](http://assets.toptal.io/uploads/blog/image/252/toptal-blog-image-1389090346415.png)

注：图片来自参考资料1

## 为什么需要 mock

在写 unittest 的时候，如果系统中有很多外部依赖，我们不需要也不希望把所有的部件都运行一遍。比如，要验证分享到微博的功能，如果每次测试的时候都要真实地把接口调用一遍，不仅效率低，制造很多垃圾数据，还可能因为外部因素导致 unittest 失败。对于有些耗时更久，或者无法简单创建测试环境的系统，真实的测试就显得更不必要。

我们只需要知道代码按照预期执行，并调用了相关的外部接口。还是拿分享到微博这个功能做例子，分享部分的伪代码可能是这样的：


    def share():
        """Share system generated message to weibo."""
        msg = generate_msg()
        weibo = get_weibo_client(user_id)
        weibo.upload(msg)

如果有一种方法，测试上面代码的时候能够运行所有的代码，但是并不实际执行 `weibo.upload(msg)`，而且还能知道每个函数被调用了几次，每次被调用的参数，那我们测试用例就方便多了。

python 中 `mock` 就是在测试的时候用来模拟外部服务的，一般下面的场景会使用到 mock：

1. 数据库操作：没有必要每一次都去读写数据库
2. HTTP 请求：网络操作很耗时，测试的时候还要依赖外部的服务
3. 外部命令：执行系统命令，比如文件操作，进程操作等等。

## Mock 的基本原理

上面也提到过，mock 是替换代码中外部的服务。因为 python 是动态语言，一切都是对象，所以在执行之前把实例、方法、函数和变量替换掉。比如

    >>> import os

    >>> def myremove(filename):
    >>>     return filename

    >>> os.remove = myremove
    <function __main__.myremove>

    >>> print os.remove('test-file')
    test-file

上面的例子是最简单的说明，如果把 `myremove` 修改成 `Mock` 类，然后这个类里面在调用的时候(复写 `__call__`)能够根据传进来的参数决定它的行为，还能记录每一次调用，你就大致了解 `Mock` 做了什么。

## Mock 的使用

### 怎么 mock 一个函数？

    from mock import Mock

    myMethod = Mock()
    myMethod.return_value = 3
    myMethod(1, 'a', foo='bar')

    myMethod.assert_called_with(1, 'a', foo='bar')    # True
    myMethod()
    myMethod.call_count                               # 2

想要 mock 出一个函数，直接使用 `mock.Mock()` 实例，你可以在初始化的时候设定返回值 `myMethod = Mock(return_value=3)`，也可以通过 `myMethod.return_value` 的属性来设置。

除了 `return_value`，你还可以 mock `side_effect`，`side_effect`是一个函数或者异常， 在 mock 的对象被调用的时候会被用同样的参数调用。

    myMethod = Mock(side_effect=KeyError('whatever'))
    myMethod()

    Traceback (most recent call last):
     ...
    KeyError: 'whatever'

上面的例子就是模拟一个异常，如果 `side_effect` 是 函数的话，这个函数就会被调用，可以用来动态地生成返回值。
下面的例子 mock 一个可以返回输入字符串长度的函数。

    def side_effect(str):
        return len(str)

    myMethod = Mock(side_effect=side_effect)
    myMethod('sd')              # 2

在 unittest 的时候，mock 还提供了下面几种 assert 语句：

+ assert_any_call
+ assert_called_once_with
+ assert_called_with
+ assert_has_calls


### 怎么 mock 一个类的方法？

要想 mock 一个类中的某个方法，可以使用 mock 提供的 `pathc` 方法：

    import mock

    import Module1

    @mock.patch.object(Module1.Class1, 'some_method')
    def test(mock_method):
        mock_method.return_value = 3
        mock_method.side_effect = some_side_effect
        m = Module1.Class1()
        m.some_method(*args, **kwargs)

        assert m.some_method is mock_method
        m.some_method.assert_called_with(*args, **kwargs)


### 怎么 mock 一个类？
有时候需要模拟一个函数或者类的行为，包括它所有的属性和方法，如果手动去一个个添加，实在低效而且容易出错。mock 提供了 `autospec` 的功能，根据提供的模板类生成一个 mock 实例。
下面是 mock 一个函数的例子，

    import mock

    def myFunc(a, b, c):
        pass


    >>> mock_func = mock.create_autospec(myFunc, return_value=3)
    >>> mock_func(1,2,3)
    >>> mock_func.assert_called_with(1,2,3)

    >>> mock_func('a string')

    Traceback (most recent call last):
     ...
    TypeError: <lambda>() takes exactly 3 arguments (1 given)

mock 一个类和这个相同：

    >>> mock_class = mock.create_autospec(myClass)


### Mock 和 MagicMock 的区别？
MagicMock 是 Mock 的扩展，允许使用 python 的 [magic methods](https://docs.python.org/3/library/unittest.mock.html#magic-methods)。例如官方提供的例子：

    >>> mock = MagicMock()
    >>> mock.__str__.return_value = 'foobarbaz'
    >>> str(mock)
    'foobarbaz'
    >>> mock.__str__.assert_called_with()

## 参考资料

+ [An Introduction to Mocking in Python](http://www.toptal.com/python/an-introduction-to-mocking-in-python)
+ [Mock official document](https://docs.python.org/3/library/unittest.mock.html)