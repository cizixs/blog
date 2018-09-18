---
layout: post
title: "linux 系统启动过程"
excerpt: " 启动了这么多次，了解一下人家会死吗?"
categories: 程序技术
tags: [linux, os, boot]
comments: true
share: true
---

## Linux 系统的启动过程

### 简介
我们都知道：操作系统运行的代码是在硬盘上的，最终要跑到内存和 CPU 上，才能被我们使用。

那从摁下电源键到看到系统界面，操作系统是怎么霸占了所有的硬件资源，把自己加载到内存开始运行的呢？
可以想到有两个可能性：操作系统自己实现的，或者有其他贵人帮忙。如果是操作系统自己启动的，就有了一个“鸡生蛋，蛋生鸡”的问题；如果是后者的话，一定有在操作系统启动之前就能工作的神力，把沉睡在硬盘的操作系统变到工作状态。

事实上，Linux 系统的启动正是上面第二种情况。帮助把 Linux 内核（Kernel）加载到内存的程序是 Boot Loader，下图的箭头表示加载关系。

> Boot Loader  ---->  Linux Kernel

现在只需要知道 `Boot Loader` 位于启动盘的第一个扇区，功能是引导 Linux 系统就可以啦，至于更详细的说明会在后面提到。那现在的问题是 `Boot Loader` 是怎么运行起来的？而且，系统怎么知道哪个可启动设备要使用呢？有时候计算机可能启动的设备可能有多个：网络、硬盘、U 盘，CD 盘等。这就需要另外一个东西来做这件事，那就是 BIOS。

> BIOS    ---->   Boot Loader 

顺着上面的思路，现在的问题是：BIOS 的谁启动的？呃，这好像是个没有止境的过程。不过幸运的是，难题就到此结束了。BIOS 是嵌在主板上的固件，计算机启动时候的约定就是启动 BIOS 开始执行。


最后总结一下 Linux 的启动过程：

1. 摁下电源键，BIOS（Basic Input/Output System）启动初始化硬件的工作，包括屏幕和键盘，内存检测，这个过程也被成为 POST（Power On Self Test），然后按照 CMOS RAM 中设置的启动设备查找顺序，来寻找可启动设备 。注：BIOS 程序嵌在主板的 ROM 芯片上的。
2. POST 过程结束后，系统的控制权从 BISO 转交到 boot loader。Boot loader 一般存储在系统的硬盘上（传统的 BIOS/MBR 系统），或者 EFI 分区上（最近的 EFI 系统）。这个时候机器不能获取外部的存储或者网络信息，一些重要的值（日期、时间、其他外部值）都是从 CMOS 里读取。CMOS 在计算机断电后也能工作的设备，Boot Loader 会在后面讲解。
3. Boot Loader 选择要启动的操作系统，加载内核镜像和初始化 RAM disk 到内存。系统的内核开始运行，直到关机为止。

![](https://preview.edge.edx.org/c4x/Linux/LFS101/asset/chapter03_flowchart_scr15_1.jpg)

### BIOS

![](https://courses.edx.org/asset-v1:LinuxFoundationX+LFS101x.2+1T2015+type@asset+block@LFS01_ch03_screen16.jpg)

摁下电源键的时候，计算机的一些寄存器被设置初值，指令寄存器 CS:IP 指向 [BIOS](https://en.wikipedia.org/wiki/BIOS) 的第一条指令。BIOS 掌握控制权，来执行硬件检测的程序，BIOS 在结束自己生命之前会寻找可启动设备。那么，BIOS 怎么知道哪些设备室可以启动的呢？如果计算机要从一个不能启动的设置加载系统，会发生严重的错误。启动设备的第一个扇区的末尾两个字节一定是：`0x55` 和 `0xAA`，这两个魔法数就是区分可启动设备和不可启动设备的关键。使用 `sudo head -c 512 /dev/sda | hd` 可以看到第一个扇区的内容，注意最后两个字符。

BIOS 如果找不到可启动设备的话，就会报`No Bootable Device Error`。


### Boot Loader

![](https://courses.edx.org/asset-v1:LinuxFoundationX+LFS101x.2+1T2015+type@asset+block@LFS01_ch03_screen18.jpg)

对于使用 BIOS/MBR 模式的系统来说，[Boot Loader](https://en.wikipedia.org/wiki/Booting) 位于硬盘的第一个扇区，也称为 MBR（Master Boot Record）。[MBR](https://en.wikipedia.org/wiki/Master_boot_record) 只有 512 字节，主要工作就是检查分区表，并找到可以启动的分区，一旦找到启动分区，就在该分区里找到后面的 Boot Loader -- 比如 [GRUB](https://en.wikipedia.org/wiki/GNU_GRUB)，把它加载到内存（RAM）。

MBR 这么有限的字节空间里，主要包括了三部分的内容：

1. bootstrap code：启动操作系统的代码
2. 分区表：指示系统盘的位置
3. 魔法数：0x55AA

![](https://courses.edx.org/asset-v1:LinuxFoundationX+LFS101x.2+1T2015+type@asset+block@LFS01_ch03_screen20.jpg)

对于使用 EFI/UEFI 模式的系统来说，UEFI 固件读取 Boot Manager 的数据来决定启动哪一个 UEFI 应用，已经找到它的位置。该固件然后启动 UEFI 应用，比如 GRUB。

现在的控制权都到了 GRUB 启动程序的手里，GRUB 根据你选择的系统（多系统的情况会有界面出现让用户选择，只有一个系统的情况会直接选择该系统），把系统的内核加载到内存开始运行，同时也会初始化 RAM disk 文件系统（[initramfs](https://en.wikipedia.org/wiki/Initramfs)）到内存，供内核使用，并把控制权交给内核。

### Linux Kernel

![](https://courses.edx.org/asset-v1:LinuxFoundationX+LFS101x.2+1T2015+type@asset+block@LFS01_ch03_screen21.jpg)

内核一般都是压缩的，所以它的首要任务是解压缩，然后检查和分析系统的硬件并初始化内核里的硬件驱动程序。内核刚加载到内存的时候，文件系统还不能使用，它使用的是 Boot Loader 加载金内存的 initramfs。
内核被加载到内存后首要工作是：初始化和配置机器的内存、处理器、存储设备等，内核也会启动一些用户态的程序。


### initramfs

![](https://courses.edx.org/asset-v1:LinuxFoundationX+LFS101x.2+1T2015+type@asset+block@LFS01_ch03_screen22.jpg)
前面提到过 boot loader 加载到内存的 RAM disk，也就是 initramfs，现在就详细讲一讲它。initramfs 包含的一些程序和二进制文件，会执行一系列的动作，保证 root 文件系统 mount 到系统。这里动作包括，为需要的文件系统提供内核的功能，以及使用 udev（User Device） 工具来发现和加载硬盘的驱动程序。 等到 root 文件系统找到后，它会检查错误然后 mount 到系统。

mount 程序告诉操作系统某个文件系统可以使用，并把它加载到文件系统的某个路径（mount point）。如果这些动作都成功的话，initramfs 就会从内存中清除，init 程序（位于 /sbin/init）开始执行。

### /sbin/init

![](https://courses.edx.org/asset-v1:LinuxFoundationX+LFS101x.2+1T2015+type@asset+block@LFS01_ch03_screen24.jpg)

init 处理挂载（mount）工作，是整个的文件系统正常运行的枢纽。需要注意的时，如果在访问存储设备的时候，需要的硬件驱动，必须在 initramfs 阶段都加载好。

到目前为止，内核程序准备好了所有的硬件资源，也把文件系统都挂载好了。它运行了 /sbin/init 程序，也就是第一个系统进程（之前运行的程序都不是 OS 级别的，不在 OS 的管辖范围），它的进程号(pid)就是 1。下面是我在自己的系统上运行 `ps aux | grep init` 的结果，第二列就是进程号：

	root         1  0.0  0.0  24320   864 ?        Ss   Jan15   0:00 /sbin/init


第一个启动的程序当然也肩负着比较重要的责任：把系统需要其他程序都启动起来。传统的 System V UNIX 工作模式下，这个过程是遍历 runlevels 序列的程序脚本，来启动或者停止预先定义的服务（service）。除了上面那个主要的任务外，init 也负责保持系统一直运行和在 shutdown 系统的清理工作，还有用户登入和登出的工作。

### 登陆和使用

前面已经说过了，init 程序负责用户的登入和登出。如果是服务器 linux 或者其他文本模式的 linux，init 就会启动 getty 程序来接受用户输入的用户名和密码来验证用户。

如果是图形界面的 linux， 会有 display manager 的服务负责检测显示屏和启动 X-server。display manager 也负责图形界面的用户登陆，以及启动正确的桌面环境。

## 参考资料

1. [edx 上 introduction to linux 的启动章节](https://courses.edx.org/courses/course-v1:LinuxFoundationX+LFS101x.2+1T2015/courseware/6cee72d455c847e9b462efb4e2dbd2a7/a73c18288e2f47d293df4ec8fbec99d1/)
2. [阮一峰介绍计算机启动的文章](http://www.ruanyifeng.com/blog/2013/02/booting.html)
3. wikipedia 上相关文章
