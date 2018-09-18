---

layout: post
title: "python collections 学习教程"
excerpt: "在某些场景，你突然发现 python 的默认四种数据类型不能满足自己的需求，来看看 collections 提供的这些结构吧。"
categories: blog
tags: [collections, dict, list, python]
comments: true
share: true
---

python 语言提供了 `list`，`tuple`，`set`，`dict`四种最常用的数据结构，能够满足绝大多数的需求，但是还有些情形需要更合适的数据结构。python 也提供了这些，放在了 `collections` 这个库。这篇文章主要介绍 `collections` 库提供的数据结构，它们的使用场景和简单的例子。

## Counter

counter 是一种特殊的字典，主要方便用来计数，key 是要计数的 item，value 保存的是个数。

    from collections import Counter

    >>> c = Counter('hello,world')
    Counter({'l': 3, 'o': 2, 'e': 1, 'd': 1, 'h': 1, ',': 1, 'r': 1, 'w': 1})

初始化可以传入三种类型的参数：字典，其他 iterable 的数据类型，还有命名的参数对。

     |  __init__(self, iterable=None, **kwds)
     |      Create a new, empty Counter object.  And if given, count elements
     |      from an input iterable.  Or, initialize the count from another mapping
     |      of elements to their counts.
     |
     |      >>> c = Counter()                           # a new, empty counter
     |      >>> c = Counter('gallahad')                 # a new counter from an iterable
     |      >>> c = Counter({'a': 4, 'b': 2})           # a new counter from a mapping
     |      >>> c = Counter(a=4, b=2)                   # a new counter from keyword args

默认请求下，访问不存在的 item，会返回 0。Counter 可以用来统计某些数据的出现次数，比如一个很长的数字串 `numbers = "67642192097348921647512014651027586741512651"` 中每个数字的频率：

    >>> c = Counter(numbers) # c 存储了每个数字的频率
    >>> c.most_common()      # 所有数字按照频率排序。如果 most_common 接受了 int 参数 n，将返回频率前n 的数据，否则会返回所有的数据
    [('1', 8),
     ('2', 6),
     ('6', 6),
     ('5', 5),
     ('4', 5),
     ('7', 5),
     ('0', 3),
     ('9', 3),
     ('8', 2),
     ('3', 1)]

此外，你还可以对两个 Counter 对象进行 `+`, `-`，`min`， `max` 等操作。

## deque

deque 是 `double-ended queue`的缩写，类似于 list，不过提供了在两端插入和删除的操作。

+ `appendleft` 在列表左侧插入
+ `popleft` 弹出列表左侧的值
+ `extendleft` 在左侧扩展

例如：

    queue = deque()
    # append values to wait for processing
    queue.appendleft("first")
    queue.appendleft("second")
    queue.appendleft("third")
    # pop values when ready
    process(queue.pop()) # would process "first"
    # add values while processing
    queue.appendleft("fourth")
    # what does the queue look like now?
    queue # deque(['fourth', 'third', 'second'])

## defaultdict
defaultdict 主要用来需要对 value 做初始化的情形。对于字典来说，key 必须是 hashable，immutable，unique 的数据，而 value 可以是任意的数据类型。如果 value 是 list，dict 等数据类型，在使用之前必须初始化为空，有些情况需要把 value 初始化为特殊值，比如 0 或者 ''。

    from collections import defaultdict

    person_by_age = defaultdict(list)
    for person in persons:
        d[person.age].append(person.name)

defaultdict 和 dict 的使用方法一样，只有在初始化的时候必须传入一个 callable 的对象 x，当访问某个还不存在的 key 时，会把 value 自动设置成 x()。比如上例中，当第一次访问某个年龄的人 `d[person.age]` 就会变成 `list()`， 也就是 `[]`。

当然也可以使用自己定义的 callable 对象，比如：

    d = defaultdict(lambda: 0)
    d["hello"] += 1         # 1
    d["a"]                  # 0


defaultdict 要比 `dict.set_default` 效率更高，使用起来也更直观和方便。

## namedtuple
namedtuple 就是命名的 tuple，比较像 C 语言中 struct。一般情况下的 tuple 是 `(item1, item2, item3,...)`，所有的 `item` 都只能按照 index 访问，没有明确的称呼，而 namedtuple 就是事先把这些 `item` 命名，以后可以方便访问。

    from collections import namedtuple


    # 初始化需要两个参数，第一个是 name，第二个参数是所有 item 名字的列表。
    coordinate = namedtuple('Coordinate', ['x', 'y'])

    c = coordinate(10, 20)
    # or
    c = coordinate(x=10, y=20)

    c.x == c[0]
    c.y == c[1]
    x, y = c

namedtuple 还提供了 `_make` 从 iterable 对象中创建新的实例：

    coordinate._make([10,20])

## OrderedDict
如同这个数据结构的名称所说的那样，它记录了每个键值对添加的顺序。

    d = OrderedDict()
    d['a'] = 1
    d['b'] = 10
    d['c'] = 8
    for letter in d:
        print letter
    # a
    # b
    # c

如果初始化的时候同时传入多个参数，它们的顺序是随机的，不会按照位置顺序存储。

    >>> d = OrderedDict(a=1, b=2, c=3)
    OrderedDict([('a', 1), ('c', 3), ('b', 2)])

除了和正常的 dict 相同的方法之外，OrderedDict 还提供了和顺序相关的操作:
    + `popitem()`: 返回最后一个插入的键值对，如果 `popitem(last=False)` 将返回第一个插入的键值对
    + `reversed`：返回一个逆序的 OrderedDict

## 参考资料

+ [python collections](http://howchoo.com/g/mtbhy2qzota/python-collections)
+ [Using defaultdict in Python](https://www.accelebrate.com/blog/using-defaultdict-python/)
+ [collections 的官方文档](https://docs.python.org/2/library/collections.html)