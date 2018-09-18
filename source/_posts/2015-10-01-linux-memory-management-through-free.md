---
layout: post
title: "通过 free 命令理解 linux 内存管理"
excerpt: "free 命令可以用来查看 linux 的内存使用情况，但是要了解输出中这些数字的含义，还要知道 linux 是怎么管理内存的。"
categories: blog
tags: [linux, free, memory]
comments: true
share: true
---

## 简介

linux 下面查看内存状态可以使用 `free` 命令，但是如果不了解 linux 内存管理机制的话，对输出也会摸不着头脑，这篇文章就说明一下各个数据的意思。

下面是我电脑上虚拟机，直接使用 `free` 命令的数据结果。

    vagrant@precise64:~$ free
                 total       used       free     shared    buffers     cached
    Mem:        374256     330952      43304          0      14400     238128
    -/+ buffers/cache:      78424     295832
    Swap:       786428       2224     784204


所有的数据默认都是 KB，第一行有六个值：

+ total：物理内存大小，就是机器实际的内存
+ used：已使用的内存大小，这个值包括了  cached 和 应用程序实际使用的内存
+ free：未被使用的内存大小
+ shared：共享内存大小，是进程间通信的一种方式
+ buffers：被缓冲区占用的内存大小，后面会详细介绍
+ cached：被缓存占用的内存大小，后面会详细介绍

其中有

+ total = used + free

下面一行，代表应用程序实际使用的内存：

+ 前一个值表示 `- buffers/cached`，即 `used - buffers/cached`，表示应用程序实际使用的内存
+ 后一个值表示 `+ buffers/cached`，即 `free + buffers/cached`，表示理论上都可以被使用的内存

不难看出来，这两个值加起来也是 total。

第三行表示 swap 的使用情况：总量、使用的和未使用的。

介绍完了这些数字，要想比较深入的了解它们的含义，还要知道其中的三个概念：cache， buffer 和 swap。

## cache

cache 就是缓存的意思。当系统读文件的时候，都是把数据从硬盘读到内存里，因为硬盘比内存慢很多，所以这个过程会很耗时。为了提高效率，linux 会把读进来的文件在内存中缓存下来（因为读取相近部分的内容是程序很常见的情况），即使程序结束，cache 也不会被自动释放。所以呢，如果有程序进行大量的读文件操作，你会发现内存使用率就上去了。

不过也不用担心，如果其他程序使用要使用内存的时候，linux 也会把这些没人使用的  cache 释放掉，给其他运行的程序使用。当然你也可以手动去释放掉这部分内存：

    echo 1 > /proc/sys/vm/drop_caches
    
## buffer

buffer 的意思和 cache 相近，不过稍有区别。考虑内存写文件到硬盘的过程，因为硬盘太慢了，如果内存要等待数据写完之后才继续后面的操作，实在是效率很低的事情，也会影响程序的运行速度。所以就有了 buffer，写到硬盘的数据会放到 buffer 里面，内存很快把数据写到 buffer，可以继续其他的工作，而硬盘可以在后台慢慢读出 buffer 中的数据，保存起来。这样就提高了读写的效率！

讲一个大家会经常遇到的例子，当我们把电脑里中的文件拷贝到 U 盘的时候，如果文件特别大，大家会遇到这种情况：明明看到文件已经拷贝完了，但系统还是会提示 U 盘正在使用中。这就是 buffer 的原因，拷贝程序把东西放到 buffer 之后，但是 U 盘还没有写完。

同样的，可以手动来 flush buffer 的内容，使用的命令是 `sync`。

## swap

swap 是实现虚拟内存的重要概念。如果系统的负载太大，内存被用完，可能会出现严重的问题。swap 就是把硬盘上一部分空间当做内存使用，正在运行程序会使用物理内存，把没有正在使用的内存放到硬盘，这叫做 swap out；而把硬盘 swap 部分的内存重新放到物理内存中，叫做 swap in。

swap 可以再逻辑上扩大内存空间，但是会造成系统变慢，因为硬盘读写速度很慢。linux 系统比较智能，会把那些不怎么频繁使用的内存放到 swap。

## 做实验验证一下

说了这么多，还是做个试验更清楚，也更有说服力。下面的试验在 virtualbox 创建的虚拟机 ubuntu 12.04 上进行的，基本的系统信息如下：

    root@precise64:/home/vagrant/test# uname -a
    Linux precise64 3.2.0-23-generic #36-Ubuntu SMP Tue Apr 10 20:39:51 UTC 2012 x86_64 x86_64 x86_64 GNU/Linux
    
最开始的时候，先查看一下内存信息（我使用了 `-m` 参数，让所有的数据都是以 M 为单位）：

    root@precise64:/home/vagrant/test# free -m
                 total       used       free     shared    buffers     cached
    Mem:           365         69        295          0          0          5
    -/+ buffers/cache:         64        301
    Swap:          767          2        765


从上面可以看到：

1. 虚拟机内存为 365M
2. 其中使用 69M，其中 64M 是真正使用的，还有 5M 是 cache

好，我们用 `dd` 命令创建一个 50M 的文件试试：

    dd if=/dev/zero of=test.log bs=1M count=50

查看一下刚创建的文件：

    root@precise64:/home/vagrant/test# ls -lh
    total 50M
    -rw-r--r-- 1 root root 50M Oct  1 09:03 test.log
    
再来看一下内存信息：

    root@precise64:/home/vagrant/test# free -m
                 total       used       free     shared    buffers     cached
    Mem:           365        121        244          0          0         55
    -/+ buffers/cache:         65        300
    Swap:          767          2        765

可以看到 cache 的内容变成了 `55M`（多出来的 `50M` 就是我们刚刚创建出来的文件大小），其他的信息都没有变化。

现在这部分文件都在内存里，我们读取的话就会很快：

    root@precise64:/home/vagrant/test# time cat test.log >/dev/null
    
    real	0m0.008s
    user	0m0.004s
    sys	0m0.008s

然后手动清理一下 cache：

    echo 1 > /proc/sys/vm/drop_caches

我们在读一次相同的文件：

    root@precise64:/home/vagrant/test# time cat test.log >/dev/null
    
    real	0m0.108s
    user	0m0.000s
    sys	0m0.040s

时间已经从刚才的 `0.008s` 变成了 `0.108s`，速度慢了很多。

再看一下内存，发现内存又恢复到原样（有 1M 的 cache 减少，可能其他的 cache 内容被刚才的命令也一同清理了）：

    root@precise64:/home/vagrant/test# free -m
                 total       used       free     shared    buffers     cached
    Mem:           365         68        296          0          0          4
    -/+ buffers/cache:         64        301
    Swap:          767          2        765
    



现在的问题来了：**如果我们创建的文件大小超过了物理内存，情况会怎么样呢？**

我们同样可以做一下实验，因为内存只有 365M，就创建一个 500M 的文件：

    dd if=/dev/zero of=test.log bs=1M count=500
    
然后来看一下内存：

    root@precise64:/home/vagrant/test# free -m
                 total       used       free     shared    buffers     cached
    Mem:           365        359          5          0          0        289
    -/+ buffers/cache:         70        295
    Swap:          767          2        765

cache 变成了 `289M`，并没有出现异常，这是因为系统会自动清理没有用的 cache 来给后面的应用。还有一点要注意，不管怎么样，cache 并不会把内存占满。

## 参考资料

+ [Memory Management 章节，节选自 Linux System Administrator Guide ](http://www.tldp.org/LDP/sag/html/buffer-cache.html)
+ [stackexchange 上关于如何释放 buffer/cache 的问答](http://unix.stackexchange.com/questions/87908/how-do-you-empty-the-buffers-and-cache-on-a-linux-system)
+ [Linux ate my ram 网站](http://www.linuxatemyram.com/)