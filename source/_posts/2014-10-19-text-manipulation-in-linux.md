---
layout: post
title: "Linux 文本处理"
excerpt: "常用的 Linux 文本处理程序"
categories: 程序技术 
tags: [text, sed, linux, bash]
comments: true
share: true
---

![KISS](http://www.faqs.org/docs/artu/graphics/kiss.png)

这篇文章记录了 Bash 中常用的文本处理的程序，但是并不对每个程序做很详细的解释，只是列出来一些实用的用法。算是读书笔记，也可以看做一个备忘。

## 1. 查看

### cat（concatenate）

+ `-A` 选项可以显示非打印字符，这个在调试 MS-DOS 的回车符时比较有用。
+ cat 主要用来查看文件和拼接文件。`cat file1.txt file2.txt file2.txt > file.txt` 会把三个文件的内容输出到`file.txt`。
+ `cat > foo.txt` 可以创建一个简单地文本编辑器，按 `Ctrl-d` 来结束输入。当然，`cat >> foo.txt` 会添加内容到文件里。
+ `cat` 还经常用在脚本里面显示多行的信息：

    cat << EOF
    Usage:
     opt1 : Do this
     opt2 : Do that
    EOF

### less 和 more

简单，不表。

### head 和 tail
+ `tail -f filename` 可以实时地查看文件最后的文本，这对于 log 文件来说很方便。

## 2. 简单处理

### sort：排序 和 uniq：重复行处理

`sort` 用来对文本进行排序，默认使用字符比较大小。

+ `sort` 可以同时排序多个文件：`sort file1.txt file2.txt file3.txt > sorted_file.txt`
+ `-n`：使用数字值进行排序
+ `-k`：`—-key=M,N` ，对 M 到 N 列之间的字符进行排序，而不是默认的整个文本行。默认使用的分隔符为 tab 和空格，可以用`-t` 来定义分隔符。`sort -t ':' -k 5 /etc/passwd | head` 对 passwd 文件里的内容根据用户名（第五项）进行排序。
+ `sort` 支持多键值进行排序，就是说可以根据多个元素进行排序。`sort --key=1,1 --key=2n file.txt` 对文件里的数据进行如下的排序： 先根据第一列进行字符排序，然后根据第二行进行数值排序。
+ `sort` 键值排序还支持偏移量的功能，如果我们要对第三列的`MM/DD/YYYY`格式的日期进行排序，可以这么做：`sort -k 3.7nbr -k 3.1nbr -k 3.4nbr distros.txt`。


`uniq` **默认只会删除相邻的重复行**，所以一般会和`sort`一起使用。
+ `-d`：输出所有重复行
+ `-c`：在开头显示重复的次数，不重复的行会显示为 1

### cut：剪切文件字符

### paste：融合文件 和 join：拼接文件

paste 读取多个文件，把每一行的文本拼接起来。如果文本行不一致，较短的文本行用空值代替。

join 和数据库的 join 功能类似，把具有相同键值的行拼接起来。

### split 分割文件

### comm/diff：比较文件 和 patch：更改文件
comm 比较两个文件，会输出三列，分别是：只出现在第一个文件的文本，同时出现在两个文件的文本和只出现在第二个文件的文本。由于功能过于简单，一般很少使用。

diff 也是比较两个文件，功能就比较强大和复杂，经常用于不同版本源码的比较。输出形式有：

+ 默认模式
+ 上下文模式，使用`-c`选项来开启
+ 统一模式，使用`-u`选项来开启

patch 程序接受 diff 的输出，把一个文件更改成另外一个文件。通常用来更新源码文件，和 git 等源码管理程序的对应程序功能一致。
`patch < diff_file` 就能把 diff 文件指明的源文件更改成目标文件。

### tr：更改字符

+ tr 主要用来“翻译”字符，就是把一个字符变成另外一个字符。它接受的两个参数分别是：要被转换的字符集，被转换后的字符集。举例，把小写字符转换成大写字符：`cat file | tr a-z A-Z`。
+ 一般情况下，两个字符集的大小相同。不同也有可能第一个字符集比第二个字符集大，比如我们要把多个字符转换为单个字符：`cat file | tr a-c A`。
+ tr 也可以用来删除字符，`-d` 选项实现这一功能。
+ tr 的 `-s` 选项还可以用来删除**相邻的重复字符**：`echo 'aaabbbccc' | tr -s ab`，这个例子会输出：`abccc`。

## 3. 更灵活高效地处理文本

### sed：Stream Editor 
sed 是强大的流编辑器，比较常用的使用方式是它的替换功能。`sed 's/hello/world/'` 能够把 `hello` 替换为 `world`。

这篇文章不会介绍 sed 的详细用法，推荐阅读*注解1*的参考资料，或者下面的文档：
+ [SED & AWK]
+ [Sed - An Introduction and Tutorial by Bruce Barnett]
+ [GUN Sed Page]
+ `man sed` 

## 注解
1. 本文根据 [The Linux Command Line 中文版] 总结而来
2. 关于各个命令的详细使用方法，请参考 `man page`
