---
layout: post
title: "docker 容器基础技术：linux namespace 简介"
excerpt: "Linux namespace 是一种内核级别的资源隔离机制，用来让运行在同一个操作系统上的进程互相不会干扰。"
categories: blog
tags: [docker, linux, kernel, container, namespace]
comments: true
share: true
---

## 1. Linux namespace 简介

Linux namespace 是一种内核级别的资源隔离机制，用来让运行在同一个操作系统上的进程互相不会干扰。

**namespace 目的就是隔离**，要做到的效果是：如果某个 namespace 中有进程在里面运行，它们只能看到该 namespace 的信息，无法看到 namespace 以外的东西。我们来想一下：一个进程在运行的时候，它会知道哪些信息？

- 看到系统的 hostname
- 可用的网络资源（bridge、interface、网络端口的时候情况……）
- 进程的关系（有哪些进程，进程之间的父子关系等）、
- 系统的用户信息（有哪些用户、组，它们的权限是怎么样的）
- 文件系统（有哪些可用的文件系统，使用情况）
- IPC（怎么实现进程间通信）
- ……

也就是说，如果要实现隔离，必须保证不同 namespace 中进程看到的上面这些东西是不同的。

如果让我来做，首先的想法是每个 namespace 都实现一整套上述资源的隔离，但是实际上 linux 的实现中，上述的所有资源都是可以单独隔离的。这遵循了 `KISS` 原则，增加了使用的灵活性（针对不同的情况使用不同的隔离自组合）。

**NOTE**：在这系列文章中，经常会遇到某些名词后面跟着带有数字的括号。这是 linux 的规范，对应的数字代表了命令所在的不同 section，可能的数字有：

1.   Executable programs or shell commands
2.  System calls (functions provided by the kernel)
3.   Library calls (functions within program libraries)
4.   Special files (usually found in /dev)
5.   File formats and conventions eg /etc/passwd
6.   Games
7.   Miscellaneous (including macro packages and conventions), e.g. man(7), groff(7)
8.   System administration commands (usually only for root)
9.   Kernel routines [Non standard]

## 2. 不同的 namespace

目前 linux 内核主要实现了一下几种不同的资源 namespace：

名称     |   宏定义  |           隔离的内容
---     |   ---     |   ----
IPC     |    CLONE_NEWIPC  |    System V IPC, POSIX message queues (since Linux 2.6.19)
Network  |   CLONE_NEWNET   |   network device interfaces, IPv4 and IPv6 protocol stacks, IP routing  tables,  firewall rules, the /proc/net and /sys/class/net directory trees, sockets, etc (since Linux 2.6.24)
Mount     |  CLONE_NEWNS    |   Mount points (since Linux 2.4.19)
PID      |   CLONE_NEWPID   |   Process IDs (since Linux 2.6.24)
User     |   CLONE_NEWUSER  |   User and group IDs (started in Linux 2.6.23 and completed in Linux 3.8)
UTS      |   CLONE_NEWUTS    |  Hostname and NIS domain name (since Linux 2.6.19)
Cgroup  |    CLONE_NEWCGROUP |  Cgroup root directory (since Linux 4.6)

这些 namespace 基本上覆盖了一个程序运行所需的环境，保证运行在的隔离的 namespace 中的，会让程序不会受到其他收到 namespace 程序的干扰。但不是所有的系统资源都能隔离，时间就是个例外，没有对应的 namespace，因此同一台 Linux 启动的容器时间都是相同的。

我们会用接下来的内容解释每个 namespace 的含义和用法。

**NOTE:** Cgroup namespace 比较新，还没有大量使用，比如 docker(v1.13.0) 目前就没有用到。

## 3. /proc 目录

每个进程都有一个 `/proc/[pid]/ns` 的目录，里面保存了该进程所在对应 namespace 的链接：

```
➜  namespace git:(uts-demo) ✗ ls -l /proc/$$/ns/
total 0
lrwxrwxrwx 1 cizixs cizixs 0 12月 21 15:36 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 cizixs cizixs 0 12月 21 15:36 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 cizixs cizixs 0 12月 21 15:36 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 cizixs cizixs 0 12月 21 15:36 net -> net:[4026531969]
lrwxrwxrwx 1 cizixs cizixs 0 12月 21 15:36 pid -> pid:[4026531836]
lrwxrwxrwx 1 cizixs cizixs 0 12月 21 15:36 user -> user:[4026531837]
lrwxrwxrwx 1 cizixs cizixs 0 12月 21 15:36 uts -> uts:[4026531838]
```

每个文件都是对应 namespace 的文件描述符，方括号里面的值是 namespace 的 inode，如果两个进程所在的 namespace 一样，那么它们列出来的 inode 是一样的；反之亦然。如果某个 namespace 中没有进程了，它会被自动删除，不需要手动删除。但是有个例外，如果 namespace 对应的文件某个应用程序打开，那么该 namespace 是不会被删除的，这个特性可以让我们保持住某个 namespace，以便后面往里面添加进程。

需要注意的是，上面列出来是我正常运行的 `bash` 程序的 namespace，并没有运行任何的容器，也没有执行任何和 namespace 有关的操作。也就是说，所有的程序都会有 namespace，可以简单理解成 namespace 其实就是进程的属性，然后操作系统把这个属性相同的进程组织到一起，起到资源隔离的作用。

正应了 David Wheeler 那句名言：

> All problems in computer science can be solved by another level of indirection.

## 4. 三个系统调用

我们知道，linux 内核提供的功能都会提供系统调用接口供应用程序使用，namespace 也不例外。和 namespace 相关的系统调用主要有三个，这部分内容就分别介绍它们的定义和使用。

**NOTE：**这些系统调用都是 linux 内核实现的，不能直接适用于其他操作系统。

### clone：创建新进程并设置它的namespace

[clone](http://man7.org/linux/man-pages/man2/clone.2.html) 类似于 fork 系统调用，可以创建一个新的进程，不同的是你可以指定要子进程要执行的函数以及通过参数控制子进程的运行环境（比如这篇文章主要介绍的 namespace）。下面是 `clone(2)` 的定义：

```
#include <sched.h>

int clone(int (*fn)(void *), void *child_stack,
         int flags, void *arg, ...
         /* pid_t *ptid, struct user_desc *tls, pid_t *ctid */ );

```

它有四个重要的参数：

- `fn` 参数是一个函数指针，子进程启动的时候会调用这个函数来执行
- `arg` 作为参数传给该函数。当这个函数返回，子进程的运行也就结束，函数的返回结果就是 exit code。
- `child_stack` 参数指定了子进程 stack 开始的内存地址，因为 stack 都会从高位到地位增长，所以这个指针需要指向分配 stack 的最高位地址。
- `flags` 是子进程启动的一些配置信息，包括信号（子进程结束的时候发送给父进程的信号 SIGCHLD）、子进程要运行的 namespace 信息（上面已经看到的 `CLONE_NEWIPC`，`CLONE_NEWNET`、`CLONE_NEWIPC`等）、其他配置信息（可以参考 `clone(2)` man page）。

### setns：让进程加入已经存在 namespace

`setns` 能够把某个进程加入到给定的 namespace，它的定义是这样的：

```
int setns(int fd, int nstype);
```

`fd` 参数是一个文件描述符，指向 `/proc/[pid]/ns/` 目录下的某个 namespace，调用这个函数的进程就会被加入到 `fd` 指向文件所代表的 namespace，`fd` 可以通过打开 namespace 对应的文件获取。

`nstype` 限定进程可以加入的 namespaces，可能的取值是：

- 0: 可以加入任意的 namespaces
- `CLONE_NEWIPC`：fd 必须指向 ipc namespace
- `CLONE_NEWNET`：fd 必须指向 network namespace
- `CLONE_NEWNS`：fd 必须指向 mount namespace
- `CLONE_NEWPID`：fd 必须指向 PID namespace
- `CLONE_NEWUSER`： fd 必须指向 user namespace
- `CLONE_NEWUTS`： fd 必须指向 UTS namespace

如果不知道 `fd` 指向的 namespace 类型（比如 fd 是其他进程打开的，然后通过参数传递过来），然后在应用中希望明确指定特种类型的 `namespace`，`nstype`  就非常有用。

更详细地说，setns 能够让进程离开现在所在的某个特性的 namespace，加入到另外一个同类型的已经存在的 namespace。

需要注意的是：`CLONE_NEWPID` 和其他 namespace 不同，把进程加入到 
PID namespace 并不会修改该进程的 PID namespace，而只修改它所有子进程的 PID namespace。

### unshare：让进程加入新的 namespace

```
int unshare(int flags);
```

`unshare` 比较简单，只有一个参数 `flags`，它的含义和 `clone` 的 `flags` 相同。`unshare` 和 `setns` 的区别是，`setns` 只能让进程加入到已经存在的 namespace 中，而 `unshare` 则让进程离开当前的 namespace，加入到新建的 namespace 中。

`unshare` 和 `clone` 的区别在于：`unshare` 是把当前进程进入到新的 namespace；`clone` 是创建新的进程，然后让新创建的进程（子进程）加入到新的 namespace。

## 5. 实践

这部分所有的代码都被放到了 [github 这个 repo](https://github.com/cizixs/namespace)，不同版本放在不同的 tag 下面，可以供读者很方便地运行。

所有的代码都在 ubuntu16.04 运行通过，操作系统信息如下：

```
➜  namespace git:(master) ✗ uname -a
Linux ubuntu-xenial 4.4.0-47-generic #68-Ubuntu SMP Wed Oct 26 19:39:52 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
➜  namespace git:(master) ✗ lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 16.04.1 LTS
Release:	16.04
Codename:	xenial
```

**NOTE:** 所有的代码都需要 root 权限执行！

### clone 系统调用

我们先来看看 `clone` 一个简单的使用例子：创建一个新的进程，并执行 `/bin/bash`，这样就可以接受命令，方便我们查看新进程的信息。后面所有的代码都是在这个框架下进行扩展的，它在 repo 中的 tag 是 `v0.1`。具体代码如下，关键部分都已经进行了注释：

```
#define _GNU_SOURCE
#include <sched.h>
#include <sys/wait.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>

// 设置子进程要使用的栈空间
#define STACK_SIZE (1024*1024)
static char container_stack[STACK_SIZE];

#define errExit(code, msg); {if(code == -1){perror(msg); exit(-1);} }


char* const container_args[] = {
    "/bin/bash",
    NULL
};

static int container_func(void *arg)
{
    pid_t pid = getpid();
    printf("Container[%d] - inside the container!\n", pid);

    // 用一个新的bash来替换掉当前子进程，
    // 这样我们就能通过 bash 查看当前子进程的情况.
    // bash退出后，子进程执行完毕
    execv(container_args[0], container_args);

    // 从这里开始的代码将不会被执行到，因为当前子进程已经被上面的bash替换掉了;
    // 所以如果执行到这里，一定是出错了
    printf("Container[%d] - oops!\n", pid);
    return 1;
}


int main(int argc, char *argv[])
{
    pid_t pid = getpid();
    printf("Parent[%d] - create a container!\n", pid);

    // 创建并启动子进程，调用该函数后，父进程将继续往后执行，也就是执行后面的waitpid
    pid_t child_pid = clone(container_func,  // 子进程将执行container_func这个函数
                    container_stack + sizeof(container_stack),
                    // 这里SIGCHLD是子进程退出后返回给父进程的信号，跟namespace无关
                    SIGCHLD,
                    NULL);  // 传给child_func的参数
    errExit(child_pid, "clone");

    waitpid(child_pid, NULL, 0); // 等待子进程结束

    printf("Parent[%d] - container exited!\n", pid);
    return 0;
}
```

**NOTE：不要忘记开头的 `#define _GNU_SOURCE`**！因为 clone 不是符合 POSIX 标准的系统调用。

这段代码不长，但是做了很多事情：

- 通过 `clone` 创建出一个子进程，并设置启动时的参数
- 在子进程中调用 `execv` 来执行 `/bin/bash`，等待用户进行交互
- 子进程退出之后，父进程也跟着退出

为了方便区分不同的进行，我们还在不同的部分打印出了父进程和子进程的 pid 以及简单的信息。来看一下具体的执行效果：

```
# 先编译代码
➜  namespace git:(master) gcc container.c -o container

# 运行程序，可以看到父进程和子进程的 pid 分别是 10666 和 10667。
# 分别在子进程和父进程中查看 /proc/$$/ns 的内容，发现两者完全一样，也就是说它们属于同一个 namespace
# 因为我们没有在 clone 的时候修改子进程的 namespace
➜  namespace git:(master) ✗ ./container
Parent[10666] - create a container!
Container[10667] - inside the container!
vagrant@ubuntu-xenial:/home/vagrant/Code/namespace$ ls -l /proc/$$/ns
total 0
lrwxrwxrwx 1 vagrant vagrant 0 Dec 21 15:53 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 vagrant vagrant 0 Dec 21 15:53 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 vagrant vagrant 0 Dec 21 15:53 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 vagrant vagrant 0 Dec 21 15:53 net -> net:[4026531957]
lrwxrwxrwx 1 vagrant vagrant 0 Dec 21 15:53 pid -> pid:[4026531836]
lrwxrwxrwx 1 vagrant vagrant 0 Dec 21 15:53 user -> user:[4026531837]
lrwxrwxrwx 1 vagrant vagrant 0 Dec 21 15:53 uts -> uts:[4026531838]
vagrant@ubuntu-xenial:/home/vagrant/Code/namespace$ exit
exit
Parent[10666] - container exited!
➜  namespace git:(master) ✗ ls -l /proc/$$/ns
total 0
lrwxrwxrwx 1 vagrant vagrant 0 Dec 21 15:54 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 vagrant vagrant 0 Dec 21 15:54 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 vagrant vagrant 0 Dec 21 15:54 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 vagrant vagrant 0 Dec 21 15:54 net -> net:[4026531957]
lrwxrwxrwx 1 vagrant vagrant 0 Dec 21 15:54 pid -> pid:[4026531836]
lrwxrwxrwx 1 vagrant vagrant 0 Dec 21 15:54 user -> user:[4026531837]
lrwxrwxrwx 1 vagrant vagrant 0 Dec 21 15:54 uts -> uts:[4026531838]

# 运行 bash 的时候，可以从提示符看到子进程的 hostname 是 `ubuntu-xenial`
➜  namespace git:(master) ✗ hostname
ubuntu-xenial
```

### UTS namespace

UTS namespace 功能最简单，它只隔离了 hostname 和 NIS domain name 两个资源。同一个 namespace 里面的进程看到的 hostname 和 domain name 是相同的，这两个值可以通过 `sethostname(2)` 和 `setdomainname(2)` 来进行设置，也可以通过 `uname(2)`、`gethostname(2)` 和 `getdomainname(2)` 来读取。

**NOTE**： UTS 的名字来自于 `uname` 函数用到的结构体 `struct utsname`，这个结构体的名字源自于 `UNIX Time-sharing System`。

代码主要修改两个地方：`clone` 的参数加上了 `CLONE_NEWUTS`，子进程函数中使用 `sethostname` 来设置 hostname。
```
static int container_func(void *hostname)
{
    pid_t pid = getpid();
    printf("Container[%d] - inside the container!\n", pid);

    // 使用 sethostname 设置子进程的 hostname 信息
    struct utsname uts;
    if (sethostname(hostname, strlen(hostname)) == -1) {
        errExit(-1, "sethostname")
    };

    // 使用 uname 获取子进程的机器信息，并打印 hostname 出来
    if (uname(&uts) == -1){
        errExit(-1, "uname")
    }
    printf("Container[%d] - container uts.nodename: [%s]!\n", pid, uts.nodename);

    ……
}
int main(int argc, char *argv[])
{
    // 把第一个参数作为子进程的 hostname，默认是 `container`
    char *hostname;
    if (argc < 2) {
        hostname = "container";
    } else {
        hostname = argv[1];
    }
    ……
    pid_t child_pid = clone(container_func,
                        container_stack + sizeof(container_stack),
                        // CLONE_NEWUTS表示创建新的UTS namespace，
                        CLONE_NEWUTS | SIGCHLD,
                        hostname); 
    ……
}
```
完整的代码在 repo 中的 tag 为 `v0.2`，感兴趣可以前往查看。下面就运行代码看看效果：

```
# 先查看父进程的 uts 信息
➜  namespace git:(master) ✗ ls -l /proc/$$/ns/uts
lrwxrwxrwx 1 cizixs cizixs 0 12月 22 11:20 /proc/21472/ns/uts -> uts:[4026531838]

# 运行 container，并把 hostname 设置为 container007
# 可以看到 container 内部 hostname 已经变了，而且对应的 uts namespace inode 也发生了改变，说明
# container 内部 uts 和父进程不一样
➜  namespace git:(master) ✗ sudo ./container container007
Parent[27312] - parent uts.nodename: [cizixs-ThinkPad-T450]!
Parent[27312] - create a container!
Container[27313] - inside the container!
Container[27313] - container uts.nodename: [container007]!
root@container007:~/Programs/namespace# hostname
container007
root@container007:~/Programs/namespace# ls -l /proc/$$/ns/uts
lrwxrwxrwx 1 root root 0 12月 22 11:18 /proc/27313/ns/uts -> uts:[4026533424]
```

此外，这个版本还新加了一个 `join_uts` 的代码，用来把进程加入到已有的 namespace。 保持上面的 container 不要退出，在另外 terminal 中运行 `join_ns` 的程序：

```
# 运行 join_ns 进入到上面 container 新建的 uts namespace
# 进程的 pid 和上面都不一样，说明是新建的进程。
# 但是 hostname 为 container007，进程的 uts inode 和上面 container 也一样
# 说明这个进程和 container 在同一个 uts namespace
➜  namespace git:(master) ✗ sudo ./join_ns /proc/27313/ns/uts
root@container007:~/Programs/namespace# hostname
container007
root@container007:~/Programs/namespace# ls -l /proc/$$/ns/uts
lrwxrwxrwx 1 root root 0 12月 22 11:23 /proc/28096/ns/uts -> uts:[4026533424]
root@container007:~/Programs/namespace# echo $$
28096
```

### PID namespace

PID namespace 隔离的是进程的 pid 属性，也就是说不同的 namespace 中的进程可以有相同的 pid。PID namespace 和我们常见的系统规则一样，都是从 `pid 1` 开始，每次 `fork`、`vfork`、`clone` 调用都会分配新的 pid。

PID namespace 第一个运行的进程的 pid 编号为 1，也被成为 init 进程。所有的孤儿进程（父进程被杀死）都会被 reparent 到 init 进程，**如果 init 进程挂掉了，系统会发送 SIGKILL 信号给该 namespace 中的所有进程来杀死它们**。由此可见，init 进程对于 PID namespace 至关重要，因此在容器中你可能听说过关于哪个程序最适合做 init 进程的争论。

PID namespace 另外一个特殊的特性是，通过 unshare 和 setns 系统调用都不会都不会把当前进程加入到新的 namespace，而是把该进程的子进程进入到里面。之所以这样设计，是因为 pid 是进程非常重要的信息，很多应用程序都会假定这个值不会变化，如果 unshare 或者 setns 把当前进程加入到新的 namespace 中，那么进程的 PID 将会发生变化，原来的 PID 也会被其他进程使用，会导致很多程序出现问题。

PID namespace 是可以嵌套的。除了 root PID namespace 之外，所有的 PID namespace 都有一个父亲 PID namespace，这个父亲 PID namespace 就是执行 `clone` 或者 `unshare` 创建该 namespace 进程所在的 PID namespace。这样的话，所有的 PID namespace 就组成了一棵树形结构。每个 PID namespace 中进程对该 PID namespace 以及所有直系祖先 PID namespace 中的进程都是可见的，而对于其他 PID namespace 都是不可见的。

既然 PID namespace 是树形组织的，那么一个进程在它可见的 PID namespace 都会有一个 pid，所有和某个进程有关的系统调用使用的 pid 都是属于调用方被创建的 PID namespace 。如果某个进程的父进程属于另外一个 PID namespace，那么调用 `getppid` 将返回 0。

![](https://uploads.toptal.io/blog/image/674/toptal-blog-image-1416487554032.png)

之前讲过，可以通过 `setns` 来修改进程所在的 namespace。对于 PID namespace 来说，进程只能移动到子 PID namespace，而不能移动到祖先 PID namespace。

对应的代码版本是 v0.3，这个版本的改动很小，只是在 `clone` 函数中添加了 `CLONE_NEWPID`。运行一下代码，测试 PID namespace 是否运行成功：

```
# 查看原来 shell 的 PID namespace
➜  namespace git:(master) ✗ ls -l /proc/$$/ns/pid
lrwxrwxrwx 1 root root 0 Aug 28 16:08 /proc/2270/ns/pid -> pid:[4026531836]

# 运行程序，进入容器内部，把容器的 hostname 改成 cizixs007
➜  namespace git:(master) ✗ ./container cizixs007
Parent[5957] - parent uts.nodename: [vagrant]!
Parent[5957] - create a container!
Container[1] - inside the container!
Container[1] - container uts.nodename: [cizixs007]!

# 查看容器内部 PID namespace 
root@cizixs007:~/code/namespace# ls -l /proc/$$/ns/pid
lrwxrwxrwx 1 root root 0 Aug 28 16:08 /proc/1/ns/pid -> pid:[4026531836]
```

可以发现，容器内部 PID namespace 文件的 inode 不同，说明进入了不同的 PID namespace。另外容器内部第一个进程的 pid 是 1，而父进程的 pid 是 `5957`，说明 PID namespace 中进程号是独立的，重新从 1 开始分配。

**NOTE：**如果在容器中运行 `ps aux` 命令，还是能看到主机上所有的进程信息，那是不是 PID namespace 隔离程度不够呢？并不是！`ps aux` 是从 `/proc` 文件中读取并展示进程信息的，这个特殊的目录并不属于 PID namespace 隔离的一部分，而是属于下面要讲到的 Mount namespace。默认情况下，容器会集成父进程的 namespace，所以 `/proc` 中还是父进程的信息，接下来我们会讲到解决方案。

### Mount namespace

Mount namespace 隔离的是 mount points（挂载点），也就是说不同 namespace 下面的进程看到的文件系统结构是不同的，namespace 内的 mount points 可以通过 `mount(2)` 和 `umount(2)` 来修改，因此Mount namespace 可以用来实现容器文件系统的隔离。

![](https://uploads.toptal.io/blog/image/677/toptal-blog-image-1416545619045.png)

- `/proc/[pid]/mounts` 文件保存了进程所在 namespace 所有已经 mount  的文件系统。
- `/proc/[pid]/mountstats` 文件保存了进程所在 namespace mount point 的统计信息
- `/proc/[pid]/mountinfo` 

因为 Mount namespace 是最早加入到 linux 的，当时并没有预计到其他 namespace 的可能性，所以它被取名为 `CLONE_NEWNS`，而不是 `CLONE_NEWMOUNT` 之类的名字。为了保持兼容性，这个名字就一直延续到现在。

通过 `clone` 和 `unshare` 创建的 Mount namespace 会默认继承父 namespace 的内容，也就是说两者看到的文件系统内容是一样的。但是之后，新 Mount namespace 的所有操作都是独立的，不会影响到其他 namespace，因此可以用来创建完全属于自己的文件系统，也可以解决之间 PID namespace 中 `/proc` 目录的问题。

Mount namespace 实现了隔离当然是好事，但是它和 PID namespace 不同，完全的隔离却带来了不便之处。因为很多时候，我们希望某个设备能自动 Mount 到所有的 namespace 中，比如系统中插入了新磁盘，虽然每个 namespace 都重新 mount 一次也能达到效果，但是未免太多复杂。

因此 Mount 引入了 shared subtree 特性，每个挂载点都有一个 `propagation type` 的属性字段，决定了 Mount 事件怎样在 Peer Group 之间传播（关于什么是 Peer Group，我们下面会解释）。这个字段有四个可选值：

- `MS_SHARED`：挂载事件（mount 和 umount）会自动传播。一个挂载点下面添加或者删除挂载点的话，同一个 peer group 的其他挂载点下面会自动执行同样的操作
- `MS_PRIVATE`：挂载信息是私有的，不共享，所有的挂载操作不属于任何 peer group，也不会影响其他任何目录
- `MS_SLAVE`：介于 `MS_SHARED` 和 `MS_PRIVATE` 之间，peer group 有 master-slave 之分，master 的挂载事件会传播到 slave；而 slave 的挂载事件不会传播到 master
- `MS_UNBINDABLE`：和 `MS_SLAVE` 类似，区别是不能作为 bind mount 的源，以防止递归嵌套。

现在来说说 peer group，它就是一个或者多个挂载点的集合，之间可以共享挂载信息。有两种情况会导致挂载点属于同一个 peer group（前提是挂载点的 propagation type 必须是 shared）：

- `mount --bind` 命令，会把源挂载点和目的挂载点放在同一个 peer group
- 创建新的 Mount namespace 时，子 namespace 和 父 namespace 也属于同一个 peer group

还有两点需要额外说明一下：

- 挂载点的父子关系是它们所在目录的父子关系决定的
- 默认情况下，子挂载点只有两种类型，如果父挂载点是 `MS_SHARED`，那么子挂载点也是 `MS_SHARED`；否则，子挂载点就是 `MS_PRIVATE` 的

来看一下代码的测试效果，对应的 tag 是 `v0.4`：

```
# 查看原来的 mount namespace inode 信息
➜  namespace git:(master) ✗ ls -lh /proc/$$/ns/mnt
lrwxrwxrwx 1 root root 0 Aug 29 09:17 /proc/2270/ns/mnt -> mnt:[4026531840]

# 运行容器应用
➜  namespace git:(master) ✗ ./container cizixs007
Parent[7436] - parent uts.nodename: [vagrant]!
Container[1] - inside the container!
Container[1] - container uts.nodename: [cizixs007]!
root@cizixs007:~/code/namespace# ls -lh /proc/$$/ns/mnt
lrwxrwxrwx 1 root root 0 Aug 29 09:17 /proc/1/ns/mnt -> mnt:[4026532183]

# ps aux 也只能看到自己 namespace 的应用
root@cizixs007:~/code/namespace# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.3  21228  3772 pts/2    S    09:17   0:00 /bin/bash
root        12  0.0  0.3  37364  3264 pts/2    R+   09:17   0:00 ps aux
```

如果回到默认 namespace 执行 `ps aux` 出现下面错误的话，是因为 `/proc` 挂载点是共享的，子 namespace 对 `/proc` 目录的重新挂载影响到了父 namespace，需要重新挂载。

```
Error, do this: mount -t proc proc /proc
```

临时的解决方案可以把 `/proc` 设置为 private，然后再运行容器程序就不会出错了：

```
$ sudo mount --make-private /proc
```

### Net namespace

Net namespace 隔离的是和网络相关的资源，包括网络设备、路由表、防火墙(iptables)、socket（ss、netstat）、 `/proc/net` 目录、`/sys/class/net` 目录、网络端口(network interfaces)等等。

一个物理网络设备只能出现在最多一个网络 namespace 中，不同网络 namespace 之间可以通过创建 veth pair 提供类似管道的通信。

要想配置一个可用的网络有需要工作要做，可以参考我之前的 [network namespace](http://cizixs.com/2017/02/10/network-virtualization-network-namespace) 这篇文章。这里就不再赘述了，我们的脚本里没有隔离网络，是为了方便，容器能够直接使用主机的网络通信。

如果添加了网络隔离，但是没有做额外的配置，那么容器中就无法和外部通信，这往往是某些应用期望的场景。比如在线代码运行平台，为了防止用户有恶意代码从网络下载脚本，或者扫描集群的网络等，使用无任何配置的隔离网络就再合适不过了。

### User namespace

User namespace 隔离的是用户和组信息，在不同的 namespace 中用户可以有相同的 UID 和 GID，它们之间互相不影响。另外，还有父子 namespace 之间用户和组映射的功能。父 namespace 中非 root 用户也能成为子 namespace 中的 root，这样就能增加安全性（如果所有 namespace 的 root 用户都是一样的，会带来子 namespace 操作父 namespace 内容的危险）。

要想实现容器内部和外部用户/组的 mapping，要把对应关系写入到 `/proc/PID/uid_map` 和 `/proc/PID/gid_map` 文件中，`PID` 表示容器内进程在父 namespace 中的 PID（如果没有 PID 隔离，也就是容器内部的 PID）。

文件内容的格式为：

```
ID-inside-ns   ID-outside-ns   length
```

- ID-inside-ns 表示 user namespace 里面用户或者组的起始 id
- ID-outside-ns 表示 user namespace 外部（也就是父 user namespace）用户或者组的起始 id
- length 表示要 mapping 的个数，如果 为 1，则表示一一映射；如果大于 1，则表示连续 mapping 多个

比如下面的内容表示将容器内 1-100 ID 映射到外部的 1000-1100：

```
1 1000 100
```

为了简单，我们的脚本也没有添加 user namespace，使用方法可以参考 LWN 网站上的文章。

### IPC namespace

IPC 可能是大家都比较少关系的一块内容，也是了解最少的只是。IPC 是进程间通信的意思，作用是每个 namespace 都有自己的 IPC，防止不同 namespace 进程能互相通信（这样存在安全隐患）。

IPC namespace 隔离的是 IPC（Inter-Process Communication） 资源，也就是进程间通信的方式，包括 System V IPC 和 POSIX message queues。每个 IPC namespace 都有自己的 System V IPC 和 POSIX message queues，并且对其他 namespace 不可见，这样的话，只有同一个 namespace 下的进程之间才能够通信。

下面这些 `/proc` 中内容对于每个 namespace 都是不同的：

- `/proc/sys/fs/mqueue` 下的 POSIX message queues
- `/proc/sys/kernel` 下的 System V IPC，包括 msgmax, msgmnb, msgmni, sem, shmall, shmmax, shmmni, and shm_rmid_forced
- `/proc/sysvipc/`：保存了该 namespace 下的 system V ipc 信息

在 linux 下和 ipc 打交道，需要用到以下两个命令：

- `ipcs`：查看IPC(共享内存、消息队列和信号量)的信息
- `ipcmk`：创建IPC(共享内存、消息队列和信号量)的信息

下面开始我们的实验，这部分对应的代码在repo 中的 tag 为 `v0.5`，读者可以跟着一起做：

```
# 先看一下全局 ipc namespace，readlink 会读取 link 的值
➜  namespace git:(master) ✗ readlink /proc/$$/ns/ipc
ipc:[4026531839]

# 先看一下系统的 hostname，以此和容器内进行区分
➜  namespace git:(master) ✗ hostname
ubuntu-xenial

# 查看系统消息队列 ipc，发现为空。然后用 `ipcmk` 创建出来一个 message queue 
➜  namespace git:(master) ✗ ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages

➜  namespace git:(master) ✗ ipcmk -Q
Message queue id: 0
➜  namespace git:(master) ✗ ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
0xeabd3aac 0          vagrant    644        0            0


# 运行我们的程序，自动创建新的 uts 和 ipc namespace
➜  namespace git:(master) ✗ sudo ./container container001
Parent[17325] - create a container!
Container[17326] - inside the container!

# 我们来看一下 ipc namespace 对应的文件，发现和之前全局 ipc namespace 不同
root@container001:/home/vagrant/Code/namespace# readlink /proc/$$/ns/ipc
ipc:[4026532134]

# 列出容器中的 message queue，发现为空，也就是说和全局 ipc 是隔离的（因为我们已经创建了全局 ipc ）
root@container001:/home/vagrant/Code/namespace# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages

# 然后使用 `ipcmk` 创建一个 message queue，注意这里的 key 和之前创建出来的也是不同的
root@container001:/home/vagrant/Code/namespace# ipcmk -Q
Message queue id: 0
root@container001:/home/vagrant/Code/namespace# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
0x0eb75307 0          root       644        0            0


# 保持上面的程序不退出，在另外一个终端运行 join_ns 程序，加入到 ipc namespace，发现可以通过 `ipcs` 看到这个 namespace 中已经创建的 message queue。证明我们使用的 ipc namespace 和上面的容器是一样的。
➜  namespace git:(master) ✗ sudo ./join_ns /proc/17326/ns/ipc
root@ubuntu-xenial:/home/vagrant/Code/namespace# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
0x0eb75307 0          root       644        0            0
```

## 6. 内核中的实现

每个进程都对应一个 `task_struct` 结构体，其中包含了 [nsproxy 变量](https://github.com/torvalds/linux/blob/master/include/linux/sched.h#L1713):

```
struct task_struct {
    ……
    volatile long state;	/* -1 unrunnable, 0 runnable, >0 stopped */
	void *stack;
	……
	
    /* namespaces */
	struct nsproxy *nsproxy;
}
```

而 nsproxy 中保存的就是指向该进程所有 namespace 的指针， `clone` 和 `unshare` 每次系统调用都会导致 nsproxy 被复制。

```
/*
 * A structure to contain pointers to all per-process
 * namespaces - fs (mount), uts, network, sysvipc, etc.
 *
 * The pid namespace is an exception -- it's accessed using
 * task_active_pid_ns.  The pid namespace here is the
 * namespace that children will use.
 *
 * 'count' is the number of tasks holding a reference.
 * The count for each namespace, then, will be the number
 * of nsproxies pointing to it, not the number of tasks.
 *
 * The nsproxy is shared by tasks which share all namespaces.
 * As soon as a single namespace is cloned or unshared, the
 * nsproxy is copied.
 */
struct nsproxy {
	atomic_t count;
	struct uts_namespace *uts_ns;
	struct ipc_namespace *ipc_ns;
	struct mnt_namespace *mnt_ns;
	struct pid_namespace *pid_ns_for_children;
	struct net 	     *net_ns;
	struct cgroup_namespace *cgroup_ns;
};
```

因为每个进程使用的 namespace 不同，因此访问得到的资源信息也不相同，从而达到资源隔离的效果。

## 7. 实用的命令：nsenter，unshare

`nsenter` 用来进入到某些指定的 namespace 中执行命令，可以方便地进行容器的调试。

ubuntu 中可以通过安装 `util-linux` 软件包获得 `nsenter`，在 ubuntu 14.04 之前，nsenter 并不存在，你可以[通过源码自己编译安装](http://askubuntu.com/questions/439056/why-there-is-no-nsenter-in-util-linux)，或者使用 [docker 版本的 nsenter](https://github.com/jpetazzo/nsenter)。

`nsenter` 常用的一个场景是进入到另外一个进程的 namespace 中进行调试，可以用下面的命令：

```
$ nseneter -t 23456 --net --pid bash
```

`-t 23456` 表示进程的 pid，`--net` 表示进入到进程的 net namespace，`--pid` 表示进入到进程的 PID namespace，`bash` 是执行的命令。这样我们就得到一个 shell，它所有的命令都是在进程的网络和 PId namespace 中执行的。更多的命令可以查看 `man nsenter`。

`unshare` 和 `nsenter` 参数有些相似，功能却不同，它是离开父进程的某些 namespace 去执行命令。

## 8. 参考资料

- [Namespaces in operation, part 1: namespaces overview](https://lwn.net/Articles/531114/)
- [Introduction to Linux namespaces - Part 1: UTS](https://blog.yadutaf.fr/2013/12/22/introduction-to-linux-namespaces-part-1-uts/)
- [Separation Anxiety: A Tutorial for Isolating Your System with Linux Namespaces](https://www.toptal.com/linux/separation-anxiety-isolating-your-system-with-linux-namespaces)
- namespaces(7) man page
- [Container's Anatomy](https://www.slideshare.net/jpetazzo/anatomy-of-a-container-namespaces-cgroups-some-filesystem-magic-linuxcon)
- [Resource Management: Linux kernel Namespaces and cgroups](http://www.haifux.org/lectures/299/netLec7.pdf)
- [Docker and the PID 1 zombie reaping problem](https://blog.phusion.nl/2015/01/20/docker-and-the-pid-1-zombie-reaping-problem/)
- [DOCKER基础技术：LINUX NAMESPACE（上）](https://coolshell.cn/articles/17010.html)