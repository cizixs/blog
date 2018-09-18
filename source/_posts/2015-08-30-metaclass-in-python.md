---
layout: post
title: "python metaclass 入门简介"
excerpt: "metaclass 一直是 python 令人害怕的怪兽，希望这篇文章能稍微阐明一些东西，让它不那么神秘和恐怖。"
categories: blog
tags: [python, metaclass, class]
comments: true
share: true
---

## 动态类型也是类型

python 是一种动态类型语言，换句话说每个变量可以在程序里任何地方改变它的类型。想要获取变量的类型信息，可以使用 `type`：

    >>> a = 2
    
    >>> type(a)
    int
    
    >>> a = '1'
    
    >>> type(a)
    str
    
    >>> type(str)
    type
    
    >>> type(type)
    type

    >>> class Test1 (object):
            pass
    
    >>> class Test2 (Test1):
            pass
    
    >>> a = Test1()
    >>> b = Test2()
    
    >>> type(a) is Test1
    True
    
    >>> type(b) is Test2
    True

从上面的例子可以得到下面的结论：

1.  每个变量都有自己的类型
2.  变量的类型是可以随时改变的
3.  `type` 只会返回对象的直接 `type`，就是定义该对象的类
4.  类的 `type` 还是 `type`，这说明 type 定义了 python 中所有类的某些基本属性


## type 的第二种用法

上面提到了 `type` 的一种用法，查看 python 对象的类型。`type` 的另外一个用法就比较独特：生成类。

    Help on class type in module __builtin__:
    
    class type(object)
     |  type(object) -> the object's type
     |  type(name, bases, dict) -> a new type

来我们看一下 type 到底怎么用！
一般情况下，类的定义式这样的：

    class Foo(object):
        """A class that does nothing."""
        def __init__(self):
            self.a = 1
        
        def magic(self):
            return self.a

使用 `type` 也可以做到同样的效果：

    def __init__(self):
        self.a = 1
        
    def magic(self):
        return self.a
        
    Foo = type('Foo', (object,), {"__doc__": "A class that does nothing.", "__init__": __init__, "magic": magic})
    
    foo = Foo()
    print foo
    print foo.a  # 1
    print foo.magic  # <bound method Foo.magic of <__main__.Foo object at 0x100fa5d50>>
    print foo.magic() # 1

`type` 的三个参数分别是：

1. name: 要生产的类名
2. bases：包含所有基类的 tuple
3. dict：类的所有属性，是键值对的字典

现在再回顾一下 “python 中一切皆对象”这句话，可能会更容易理解。


## metaclass 就是类的类
我们在前面看到怎么使用 `type` 来动态创建类，其实在 python 内部也进行着同样的步骤。这就是 metaclass 的概念！

想弄明白 metaclass，我们要搞清楚 class。因为类似于 class 定义了 instance 的行为，metaclass 则定义了 class 的行为。可以说，class 是 metaclass 的 instance。

instance 创建的时候会调用 `__init__` 函数，类似的，class 创建的时候会调用 `__new__` 函数。通过自定义 `__new__` 函数，你可以在创建类的时候做些额外的事情：把这个类注册到某个地方作为记录或者后续的查询，给类注入一些属性，或者干脆替换成另外一个类。

当遇到类定义的时候，

    class MyClass(object):
        pass

不会立即去创建这个类，而是把这段代码当做正常的 code block 来执行，结果就是生成一个命名空间(namespace)，**就是包含了要生成类(class-to-be)所有属性的字典，然后才会调用 `__new__` 函数，把类名、类的父类、类的属性传递过去，生成这个类。**

看到没有，上面一段话是不是很熟悉，没错，和我们使用 `type` 来创建类是一样的。

总结一下：metaclass 的功能是什么？根据类的信息，来创建这个类。没错，`type` 就是 python 自带的 metaclass。

## 创建自己的 metaclass
既然可以使用 `type` 来创建 python 标准的类，自然而然地，继承 `type` 这个类，覆盖已有的内置函数，就可以创建自己的 metaclsss。

下面的内容都会基于一个这样的并metaclass：它为要创建的类自动添加一个属性 `__cizixs`。这个 metalcss 没有任何实际的意义，但可以帮助理解 metaclss 。

根据上面的思路，我们可以写出这样的代码：

    class MyMetaclass(type):
        def __init__(cls, name, bases, attrs):
            cls.__cizixs = "Don't panic"
            print("In MyMetaclass for {}".format(name))
            super(MyMetaClass, cls).__init__(name, bases, attrs)
            
其实还可以覆写 `__new__` 函数来达到相同的目的：

    class MyMetaclass(type):
        def __new__(cls, name, bases, attrs):
            attrs['__cizixs'] = "Don't panic"
            print("In MyMetaclass for {}".format(name))
            return super(MyMetaclss, cls).__new__(cls, name, bases, attrs)
    
    class Foo(object):
        __metaclass__ = MyMetaclass
        
        pass
        
    #  In MyMetaclass for Foo
    
    foo = Foo()
    print foo.__cizixs  # Don't panic

关于 `__new__` 和 `__init__` 的不同，就不在此讲述，感兴趣可以自行搜索。只想提一句，`__new__` 创建类，`__init__` 初始化类。所以 `__new__` 会在 `__init__` 之前被调用，而且 `__new__` 必须要返回新创建的类，`__init__` 不需要。
 
上面这段代码需要说明的几点：

+ `__new__` 函数就是类创建的时候可以使用的
+ 注意 `__new__` 是一个 classmethod，代码中参数 `cls` 在这里就是 `<class '__main__.MyMetaclass'>`。
+ 只要在类的定义里指定 `__metaclss__`，那么这个类在创建的时候就会自动使用自定义的 metaclass，而不是系统默认的 `type`。
+ 在创建类的时候，python 会按照这样的顺序去找要使用的 metaclass：
    + 类定义时候的类变量 `__metaclass__` 
    + 类所在`MODULE`（就是当前文件）里的 `__metaclss__` 变量
    + 第一个父类定义的 `__metaclas__` 变量
    + 系统默认的 `type` metaclass

如果要控制类实例化 instance 的行为，还可以覆写 `__call__` 函数。

## 参考文档

+ [What is a metaclass in Python?](http://stackoverflow.com/questions/100003/what-is-a-metaclass-in-python/6581949#6581949)
+ [Metaclasses Demystified ](http://web.archive.org/web/20120503014702/http://cleverdevil.org/computing/78/)
+ [understanding python metaclasses](http://blog.ionelmc.ro/2015/02/09/understanding-python-metaclasses/)
+ [python metaclasses by example](http://eli.thegreenplace.net/2011/08/14/python-metaclasses-by-example)
