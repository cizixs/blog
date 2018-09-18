---
layout: post
title: "aufs 简介以及在 docker 中的使用"
excerpt: "AUFS 的全称是 Advanced Multi-layered unification filesytem，它的主要功能是：把多个目录结合成一个目录，对外使用。"
categories: blog
tags: [docker, container, aufs, storage, linux, filesystem]
comments: true
share: true
---

## AUFS 简介

AUFS 的全称是 Advanced Multi-layered unification filesytem，它的主要功能是：把多个目录结合成一个目录，对外使用。

把多个目录 mount 成一个，那么读写操作是怎样的呢？

- 默认情况下，最上层的目录为读写层，只能有一个；下面可以有一个或者多个只读层
- 读文件，打开文件的时候使用了 `O_RDONLY` 选项：从最上面一个开始往下逐层去找，打开第一个找到的文件，读取其中的内容
- 写文件，打开文件时用了 `O_WRONLY` 或者 `O_RDWR` 选项
    - 如果在最上层找到了该文件，直接打开
    - 否则，从上往下开始查找，找到文件后，把文件复制到最上层，然后再打开这个 copy（所以，如果要读写的文件很大，这个过程耗时会很久）
- 删除文件：在最上层创建一个 whiteout 文件，`.wh.<origin_file_name>`，就是在原来的文件名字前面加上 `.wh.` 

## aufs 简单实验

Ubuntu 系统默认已经安装了 aufs，对应的安装包是 `aufs-tools`。下面我们就做一个简单的试验，看看 aufs 具体的样子。

工作目录可以随便选择，后面的操作都是在这个目录进行的。首先创建三个子目录: 

- `base` 作为底层的目录
- `top` 作为上层的目录
- `mnt`： aufs 使用的挂载点，会把上面两个目录挂载到这里

然后创建几个文件，如下：

```
➜  tree                     
.
├── base
│   ├── common.txt
│   └── hello.txt
├── mnt
└── top
    ├── common.txt
    └── foo.txt
```

接下来使用 aufs，把 `base` 和 `top` 一起 mount 到 `./mnt` 目录：

```
➜  sudo mount -t aufs -o br=./top:./base none ./mnt
```

在 aufs 中，`base/` 和 `top/` 被称为 branch，它们就是源目录。

这个 mount 命令的参数意义是这样的：

- `-t aufs`：mount 的文件类型，使用的是 aufs
- `-o`：传递个 aufs 的选项，每个文件类型的选项不同
- `br`：表示 branch，也就是 aufs 需要的的各个目录
- `none`：这个本来是设备的名字，但是我们并没有用到任何设备，只会用到文件夹，因此这里为 none
- `./mnt`：挂载点，也就是内容最终出现的目录

默认情况下，`-o` 后面的第一个目录是以可读写模式挂载的，剩下的目录都是只读模式（和 docker 容器模型非常一致）。

查看挂载好之后的组织形式，发现 `./mnt` 中出现了原来两个文件夹的综合内容，其中 `common.txt` 文件选择的是 `top/` 文件夹的。

```
➜  tree
.
├── base
│   ├── common.txt
│   └── hello.txt
├── mnt
│   ├── common.txt
│   ├── foo.txt
│   └── hello.txt
└── top
    ├── common.txt
    └── foo.txt

➜  cat mnt/common.txt 
top
```

如果要修改 `common.txt` 文件，会发现只有 `top` 目录对应的内容发生了变化，`base` 下面的内容会保持不动：

```
➜  echo changed > ./mnt/common.txt
➜  cat top/common.txt 
changed
➜  cat base/common.txt 
base
```

这是因为 aufs 会逐层去查找文件，发现最上层存在文件 `common.txt` 并且是可写的，就会直接操作这个文件。类似的，如果是修改 `foo.txt` 也会直接反应在 `top/` 目录里面。

但是如果我们想要修改 `hello.txt` 文件，和预期不一样的是，`base/hello.txt` 并没有变化，而是新建了一个 `top/hello.txt` 文件，所有的操作都是在这个文件进行的。实验结果如下：

```
➜  echo hello, world > mnt/hello.txt 
➜  tree
.
├── base
│   ├── common.txt
│   └── hello.txt
├── mnt
│   ├── common.txt
│   ├── foo.txt
│   └── hello.txt
└── top
    ├── common.txt
    ├── foo.txt
    └── hello.txt
```

这是因为，aufs 从上往下查找文件，虽然在 `base/` 中发现了 `hello.txt` 文件，但是这个 branch 是以只读的方式挂载的，所以 aufs 并不能直接修改它，而是把它拷贝一份到上层，并对这个拷贝进行修改。

当然我们可以在 mount 的指定每个 branch 的读写模式，比如把两个 branch 都以可写的方式挂载：

```
➜  sudo mount -t aufs -o br=./top=rw:./base=rw none ./mnt
```

那么修改文件的规则会发生一些变化，文件查找还是从前到后，但是一旦发现文件，就能直接修改这个 branch 的文件内容，而不需要进行拷贝了。具体的实验就不做了，操作也非常简单，读者可以自行完成。

可以指定的权限一共有三种：

- `rw`：可读可写，用户能直接修改这个 branch 的文件内容
- `ro`：只读，用户不能通过 aufs 的接口对文件进行写操作，只能读取里面的内容
- `rr`：real read only，底层的文件本来就是只读的（这种情况比较少见），这种情况下，aufs 就不用担心文件不通过它的接口被修改的情况

除了读写模式之外，还有一个重要的属性——`whiteout`。

通过 aufs 指定的读写模式，只有用户通过最终的挂载点访问才有效，如果用户绕过挂载点，直接修改原来的文件，aufs 应该怎么处理呢？这个行为是由一个参数控制的，`udba`（全称是 User Direct Branch Access），这个参数有三个可选值：

- `udba=none`：aufs 不会进行任何数据同步的检查，因此性能会高一点，但是可能会出现数据不一致的情况。如果用户能保证文件不会直接被修改，或者对文件内容一致性要求不高，可以使用
- `udba=reval`：aufs 会检查底层的文件有没有改动，如果有的话，把改动的内容更新到挂载点。这个性能会导致 aufs 产生额外的性能损耗
- `udba=notify`：通过 inotify 监听底层的文件变化，基于事件驱动，能够减少第二种方式的性能损耗

说了这么多，可以看出来其实 aufs 最核心的功能还是那句话：**把多个目录合并成一个目录，让用户决定在操作统一的文件系统**。虽然看起来很有趣，那么 aufs 有哪些实际的用处呢？当然它被我们提起是因为 docker 可以用它来保存镜像和容器，但是 aufs 出现的时间要比 docker 长很多，它常见的用法包括：

- Linux 光盘演示和教程，录制了 Linux 的光盘可以用来让用户体验，但是光盘的内容是只读的，可以通过 aufs 把光盘和 U 盘或者磁盘 mount 到一起，用户对文件的修改保存到后面的存储上
- 如果系统上因为各种原因，不同用户的 home 目录保存在不同的路径和磁盘上，可以通过 aufs 把它们 mount 到一起，统一进行操作

当然，下面我们就要讲讲 aufs 在 docker 中的用法。

## docker 使用 aufs 作为 storage driver

在 ubuntu 系统中，安装了 docker 之后，docker 运行默认选择的 storage driver 就是 aufs，通过 `docker info` 命令可以查看，我自己的机器上显示的信息如下：

```
➜  docker info
Server Version: 17.03.0-ce
Storage Driver: aufs
 Root Dir: /var/lib/docker/aufs
 Backing Filesystem: extfs
 Dirs: 228
 Dirperm1 Supported: true
```

所有 aufs 的内容都在 `/var/lib/docker/aufs` 目录中，这个目录下面有三个子目录：

```
➜ tree -L 1 /var/lib/docker/aufs
/var/lib/docker/aufs
├── diff
├── layers
└── mnt
```

- `diff`：镜像每一层的内容，每个文件夹代表一个层
- `layers`：镜像各层是怎么组织的 metadata，每个文件代表一个层，这个文件中保存着它下面所有层的 ID（按照顺序）
- `mnt`: 镜像或者容器的 mountpoints，也就是容器最终看到的文件系统的样子

通过这三个子目录，docker 就能实现镜像的分层存储、容器的 Copy-On-Write 启动。

## docker 容器的文件存储解析

先来看看镜像，docker 的镜像是分层的，而且这些层之间有父子关系，它们共同组成了我们看到的一个个镜像。在本地，这些层是保存在 `/var/lib/docker/aufs/diff` 目录下的，我们可以用 `docker inspect ubuntu:16.04` 查看 `ubuntu:16.04` 有哪些层：

```
"RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:90edd0ba21c8da7e530c3fdb0af496a07a33c285c7e51f30de80c50c624a5905",
                "sha256:267964ef478ec7e5969fc9c6efa41026195bc9bc4c6d6a06aa319adbd4378b5c",
                "sha256:bec30309c6f4462637b06947692a17fd3e3ba6a0233f74c7c9292b4930421541",
                "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef",
                "sha256:4a3596d391da67de46a4f50b07f69277e4c81d65debdf68b99aa726959602e39",
                "sha256:72a672688aec3c93f4a1c6af75494c163347ad5319a582fe01c435bc84b08295",
                "sha256:11d4787bae4222ff2790dc6d9678d8c205286b86c33cad3ec80762602799384c",
                "sha256:0e593a4c1af6701dd33b58fead3fdb276cd2b87f020085297e8f690316e61b85",
                "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef",
                "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef"
            ]
        }
```

可以看到，`ubuntu:16.04` 一共有 10 层，每一层都对应了 `aufs/diff` 下面的一个目录，但是每个目录名和上面的 `sha256` ID 并不相同。

接下来，我们运行容器：

```
$ docker run -d ubuntu:16.04 sh -c "while true; do sleep 1; echo 1; done"
```


这个容器会在 `aufs/mnt` 目录下创建一个目录：`/var/lib/docker/aufs/mnt/223993596e6e22217d86604b374e24d973ebe48254f34a92a5e960d4e3860caa`，这也是容器中最终看到的文件系统的内容。

怎么中找到每个容器的层级关系呢？可以先通过通过 `cat /proc/mounts` 看到 aufs 的内部 ID(si)，比如下面的 `bca8de84a45d534b`：

```
➜  cat /proc/mounts| grep aufs
none /var/lib/docker/aufs/mnt/223993596e6e22217d86604b374e24d973ebe48254f34a92a5e960d4e3860caa aufs rw,relatime,si=bca8de84a45d534b,dio,dirperm1 0 0
```

然后就能根据这个 id，查看它保存的各个 branch 的信息（对应了 docker 的每个层）：

```
➜  cat /sys/fs/aufs/si_bca8de84a45d534b/br[0-9]*
/var/lib/docker/aufs/diff/223993596e6e22217d86604b374e24d973ebe48254f34a92a5e960d4e3860caa=rw
/var/lib/docker/aufs/diff/223993596e6e22217d86604b374e24d973ebe48254f34a92a5e960d4e3860caa-init=ro+wh
/var/lib/docker/aufs/diff/0809f5e8e6753bce10504289fceed0c377c6cc99e2a8d66505c56f01b85217a3=ro+wh
/var/lib/docker/aufs/diff/5febf4aecb9a60edb6d789da279490e3677f40b2e5af3bdfeec51bd2d1bef230=ro+wh
/var/lib/docker/aufs/diff/ec53b179c0bf8a9f9d729e19ca9ecbc7230d4c5daa7bf88bc2ef049a6c939800=ro+wh
/var/lib/docker/aufs/diff/2d1b95d7488f268999bfe058ca114c2efdbc0772f57ee7f6c96bbeb05577f2db=ro+wh
/var/lib/docker/aufs/diff/55d5a8091dc476bf0f4e39a119408acb4b8b690e3ace7022e7aaeea30b404d20=ro+wh
/var/lib/docker/aufs/diff/893023d4e2be28ba1660dd035bd0b3892478c50c83d2c8d6477fa9170068d2e4=ro+wh
/var/lib/docker/aufs/diff/f876a0ffdfa44a939b8b851f6b2cce086836acff6d55701739b2ad8d786ae346=ro+wh
/var/lib/docker/aufs/diff/e1863e30cbc5198d0c84f6611c0f5b2a6750abcefdb29aa57c4c4cd0becadd54=ro+wh
/var/lib/docker/aufs/diff/dcc613560b91ab550919a7f5b89025b1144ee63fa6dcdab929dec237ef3e8280=ro+wh
/var/lib/docker/aufs/diff/b671e25db34edc80d115c50338c9546faad8abf704c1d5c1450a9d9a7d84e8b6=ro+wh
```

可以看到，除了第一个是 `rw` 之外，其他都是 `ro+wh`（`ro` 表示 readonly 只读，`wh` 表示 whiteout，目录中可以包含 whiteout 文件），你可以查看各个目录的内容，它们对应了每个层的修改。

![](https://docs.docker.com/engine/userguide/storagedriver/images/aufs_layers.jpg)

最终看到的文件是这样的：

```
$ ls aufs/mnt/223993596e6e22217d86604b374e24d973ebe48254f34a92a5e960d4e3860caa     
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

因为我运行的容器并没有修改任何文件，因此最上面的层最开始为空。在容器中创建一个 `/root/hello.txt` 文件，这个目录就能看到新创建的文件：

```
➜  ls aufs/diff/223993596e6e22217d86604b374e24d973ebe48254f34a92a5e960d4e3860caa
➜  tree aufs/diff/223993596e6e22217d86604b374e24d973ebe48254f34a92a5e960d4e3860caa/
aufs/diff/223993596e6e22217d86604b374e24d973ebe48254f34a92a5e960d4e3860caa/
└── root
    └── hello.txt

1 directory, 1 file
```

同理，直接在这个目录中新建文件也能在容器里看到。

## 总结

aufs 是 docker 最早选择的存储驱动，因为 docker 公司最开始支持的操作系统为 Debian。aufs 概念很简单，非常容易理解和使用，但是 aufs 也有它的问题。

最大的问题是它不没有进入到 Linux 内核，因此不能保证可移植性，虽然像 ubuntu 这种发行版默认支持 aufs，但并不是所有系统都如此，比如 centos 就默认没有提供 aufs 支持。之所以没有合并到内核，据说是因为 aufs 的实现代码很冗杂，Linus 认为代码质量太差，虽然开发者多次精简，最终还是没有进入到内核，而且在短时间内也不会有什么变化。

另外的问题是 aufs 对性能有比较大的影响，通过上面的知识，我们至少能看到两个性能问题：第一次修改一个大文件会非常耗时；另外层级过多也会影响整体的读写性能。此外，[这里有篇文章](https://sthbrx.github.io/blog/2015/10/30/docker-just-stop-using-aufs/)提出，aufs 在频繁打开文件的时候性能损耗很大，虽然只是一个例子，而且没有给出 root cause，但能从另外一个侧面反应出 aufs 性能确实有待商榷。

如果在生产环境适用，上面两点因素会使很多人不会选择 aufs。但是 aufs 非常适合在开发环境，或者对性能要求较低的情况，因为 ubuntu 默认的驱动就是 aufs，而且它确实足够简单，维护的压力会小很多。

## 参考资料

- [DOCKER基础技术：AUFS](https://coolshell.cn/articles/17061.html)