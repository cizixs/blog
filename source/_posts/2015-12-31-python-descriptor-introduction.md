---
layout: post
title: "python 描述器简介"
excerpt: "python 中的描述器（descriptor）是很多内部方法实现的秘密，理解了它就能对面向对象编程有更深一步的理解。和 metaclass 一样，虽然不常用，但是在阅读很多源码的时候还是会经常遇到的。"
categories: blog
tags: [python, descriptor, class]
comments: true
share: true
---


## 什么是 descriptor

python 的描述器是在 python 2.2 版本引入的一个特性，那么我们要搞清楚它添加进来要解决什么样的问题。

有时候，我们希望对对象的属性有更强的控制：比如希望某个值在一定的范围内（比如温度，年龄等），或者希望赋值的时候要是某个类型的值，再比如希望某个值根据另外的属性值动态地调整（表示身体健康状况的属性要根据体温变化）。如果你想到了 python 的 property 装饰器，很好！不过 property 的内部就是用描述器实现的，而且，如果我们希望属性是通用的，不仅仅依附于某个特定的类，这时候 property 就不能满足需求了。

描述器的功能能强大，python 内部的类方法，前面提到的 property，还有static method 和  classmethod 都是描述器实现的。这篇文章后面也会简单分析这些特性。

说简单点，描述器就是把类的某个属性转换成一个特殊类，访问这个属性的时候会调用这个特殊类的某些内部函数，来达到灵活控制属性的目的。可能说的有些玄乎，还是继续往下看吧。

## descriptor 的定义和使用

前面说了，描述器是一个特殊的类，如果某个类定义了下面这些方法的任意一个或者多个，那么它就是一个描述器：

+ `__get__(self, instance, owner)`：获取这个属性的值，返回属性的值或者抛出 AttributeError 异常
+ `__set__(self, instace, value)`：设置这个属性的值，没有返回值
+ `__delete__(self, instance)`：删除这个属性，也没有返回值

举个简单的例子，我们来写个姓名属性的描述器：

    class NameProperty(object):

        def __init__(self):
            self._name = ''

        def __get__(self, instance, owner):
            print("Getting {}".format(self._name))
            print instance, owner
            return self._name

        def __set__(self, instance, name):
            print("Setting {}".format(name))
            if not isinstance(name, string):
                raise TypeError("name must be a string, but got {}".format(type(name))
            self._name = name.title()

        def __del__(self, instance):
            print("Deleteing {}".format(self._name))
            del self._name


    class Person(object):
        name = NameProperty()
        age = 23

然后就可以调用这个类：

    In [23]: p = Person()

    In [24]: p.name
    Getting
    <descriptor.Person object at 0x10f09a8d0> <class 'descriptor.Person'>
    Out[24]: ''

    In [25]: p.name = 'cizixs'
    Setting cizixs

    In [26]: p.name
    Getting Cizixs
    <descriptor.Person object at 0x10f09a8d0> <class 'descriptor.Person'>
    Out[27]: 'Cizixs'

这个例子中，我们简单地把用户赋值的 name，转换了大小写，并且保证赋值的名字是字符串。可以看到，当我们使用 `p.name` 的时候，实际上调用的是我们之前定义的函数。其中传过去的两个参数 instance 就是实例（这里的 p），owner 就是定义的类（这里的 Person）。

如果一个类同时定义了 `__get__()` 和 `__set__()` 方法，我们称之为数据描述器（data descriptor）；如果只定义了 `__get__()` 方法，我们称之为非数据描述器（non-data descriptor），python 内部的 static method 和 classmethod 都是后者。

需要注意的是：**描述器是赋值给类的，而不是实例的。**

## descriptor 的调用顺序
我们知道当访问实例 `a` 属性 `x` 的时候，python 会先查看 `a.__dict__['x']`，然后会访问 `type(a).__dict__['x']`，然后依次访问 `type(a)` 的基类。

当我们调用 `obj.x`的时候，如果 `x` 是描述器，会根据 obj 是对象还是类有不同的调用顺序：

如果是对象，自动访问是在 `obj.__getattribute__()` 函数中完成的，这个函数会把 `a.x` 转化成 `type(a).__dict__['x'].__get__(a, type(a))`。这个调用的优先级如下：

1. 首先调用数据描述符（如果定义了的话）
2. 其次调用实例变量
3. 然后是非数据描述符（如果定义了的话）
4. 最后是 `__getattr__` 内部函数（当以上调用都没有返回的时候）

具体的 CPython 实现可以在 `Objects/object.c` 文件的 `PyObject_GenericGetAttr()` 函数中找到。

如果是类，那么自动访问是在 `type.__getattribute__()`，它会把 `A.x` 转换成 `A.__dict__['x'].__get__(None, A)`，描述器的值会覆盖掉类的属性值，用 python 代码可以近似表示为：

    def __getattribute__(self, key):
        "Emulate type_getattro() in Objects/typeobject.c"
        v = object.__getattribute__(self, key)
        if hasattr(v, '__get__'):
           return v.__get__(None, self)
        return v

总结一下，需要记住以下几点：

+ 描述器是在 `__getattribute__` 内部方法中被调用的
+ 覆写 `__getattribute__` 可以防止描述器被自动调用
+ 调用优先级：数据描述器 > 实例变量 > 非数据描述器

## python 内部的 descriptor

其实在 python 内部有很多描述器的用处，下面会依次介绍一下。

### 属性：property

python 提供 property 来把自定义的方法变成属性的 getter 和 setter，比如：

    class C(object):
        def getx(self): return self.__x
        def setx(self, value): self.__x = value
        def delx(self): del self.__x
        x = property(getx, setx, delx, "I'm the 'x' property.")

此外，还提供了装饰器来简化这个过程，比如上面的代码也可以写成：

  class C(object):
      @property
      def x(self): return self.__x

      @x.setter
      def setx(self, value): self.__x = value

      @x.deleter
      def delx(self): del self.__x

  其实这很容易通过描述器实现，python descriptor HOWTO 官方教程中就给出了如下的代码：

      class Property(object):
        "Emulate PyProperty_Type() in Objects/descrobject.c"

        def __init__(self, fget=None, fset=None, fdel=None, doc=None):
            self.fget = fget
            self.fset = fset
            self.fdel = fdel
            if doc is None and fget is not None:
                doc = fget.__doc__
            self.__doc__ = doc

        def __get__(self, obj, objtype=None):
            if obj is None:
                return self
            if self.fget is None:
                raise AttributeError("unreadable attribute")
            return self.fget(obj)

        def __set__(self, obj, value):
            if self.fset is None:
                raise AttributeError("can't set attribute")
            self.fset(obj, value)

        def __delete__(self, obj):
            if self.fdel is None:
                raise AttributeError("can't delete attribute")
            self.fdel(obj)

        def getter(self, fget):
            return type(self)(fget, self.fset, self.fdel, self.__doc__)

        def setter(self, fset):
            return type(self)(self.fget, fset, self.fdel, self.__doc__)

        def deleter(self, fdel):
            return type(self)(self.fget, self.fset, fdel, self.__doc__)

### 函数 VS 方法
有了描述器的知识，我们就能更清楚地明白 python 中的函数和方法的区别，以及 bound 和 unbound 到底是怎么一回事了。

先来看一下例子：

    class Bar(object):
        def __init__(self, name):
            self.name = name

        def pname(self):
            print self.name

    In [71]: Bar.__dict__['pname']
    Out[71]: <function __main__.pname>

    In [72]: b.pname
    Out[72]: <bound method Bar.pname of <__main__.Bar object at 0x10f102550>>

    In [73]: Bar.pname
    Out[73]: <unbound method Bar.pname>

我们看到，在 `Bar.__dict__` 存储的 `pname` 其实是个函数（function），到了 `Bar.pname` 变成了 unbound method，在 `b.pname` 有变成了 bound method。这个到底是怎么回事呢？

如果从描述器这个视角来看，就清楚很多：

+ 当我们使用 `b.pname` 时候，因为 pname 是描述器，python 内部会调用 `pname.__get__(self, b, Bar)` 返回一个 bound method
+ 当我们使用 `Bar.pname` 的时候，python 调用 `pname.__get__(self, None, Bar)`，返回一个 unbound method

一个近似的实现是（也是官方 HOWTO 提供的）：

    class Function(object):
        . . .
        def __get__(self, obj, objtype=None):
            "Simulate func_descr_get() in Objects/funcobject.c"
            return types.MethodType(self, obj, objtype)

而 self 在这里就是为了把实例传到特定的函数而定义的关键字。

### 静态方法和类方法：staticmethod 和 classmethod
类似的，staticmethod 会完全忽略 instance 和 owner 变量，而直接返回之前定义的函数：

    class StaticMethod(object):
     "Emulate PyStaticMethod_Type() in Objects/funcobject.c"

       def __init__(self, f):
            self.f = f

       def __get__(self, obj, objtype=None):
            return self.f

classmethod 会把 owner 或者 type(instance) 传给原来的函数作为第一个参数 klass：

    class ClassMethod(object):
         "Emulate PyClassMethod_Type() in Objects/funcobject.c"

         def __init__(self, f):
              self.f = f

         def __get__(self, obj, klass=None):
              if klass is None:
                   klass = type(obj)
              def newfunc(*args):
                   return self.f(klass, *args)
              return newfunc

## 其他用法

描述器另外一个比较常见的用法是某些属性的缓存：

  class cached_property(object):
      def __init__(self, func):
          self.func = func

      def __get__(self, obj, cls):
          value = obj.__dict__[self.func.__name__] = self.func(obj)
          return value

使用起来也比较简单：

  class Foo(object):

    @cached_property
    def hello(self):
      return calculate_value()

如果某个属性初始化的时候需要计算，比如上面的 `calculate_value`，这个描述器只有在第一次使用的时候去计算，
然后把结果存到 `__dict__`（名字和方法名一样），下次再访问的时候，就会优先访问 `__dict__` 里面的值。


## 参考文档

+ [Descriptor HowTo Guide](https://docs.python.org/2/howto/descriptor.html)
+ [Classes and Objects II: Descriptors](http://intermediatepythonista.com/classes-and-objects-ii-descriptors)
