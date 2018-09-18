---
layout: post
title: "Sublime Text: python 程序员不完全指南"
categories: 程序技术
excerpt: "配置 sublime 确实比写代码更有成就感!"
tags: [Sublime]
comments: true
share: true
---

## 1. 插件

### Package Control

唯一需要手动安装的插件，官方介绍：

> The Sublime Text package manager that makes it exceedingly simple to find, install and keep packages up-to-date.

安装的话[官网上有自动安装脚本](https://sublime.wbond.net/installation)。

![sublime text](https://raw.githubusercontent.com/cizixs/cizixs.github.io/master/images/sublime_text.png)

### Git
Git 和 Sublime 的深度集成，直接执行 Git 命令，轻松告别 Terminal。

### GitGutter
实时查看 git diff 信息，在文件右侧的行号前添加记号来显示内容的增删改。

###  SublimeBlockCursor
把光标变成 Block，这样就不会发愁找不到当前光标的位置啦！

### AllAutoComplete
在所有打开的文件中搜索词库来自动补全

### AutoPEP8
自动实现 python PEP8 的代码规范，必备。

### Pylinter
更严格的代码静态检查，设置有点复杂，不建议初级 Python 用户安装。

### SublimeREPL

NodeJS, Python, Ruby, Scala and Haskell 等语言的解释器，有语法问题就能直接测试。

### Advanced new file
智能的新建文件选项

### Alignment
自动对齐插件，代码洁癖患者必备。

---------

### DocBlockr
Javascript, PHP, CoffeeScript, Actionscript, C & C++ 语言的自动文档填充

### Emmet
代码自动快速扩展，用于 HTML/CSS 前端开发

## 2. 快捷键


### 大小写转换：⌘ + K 然后 U, ⌘ + K 然后 L



快捷键             |           功能              |   详情
--------        |   -----------
⌘ + ⇧ + p       | 终极杀手      |   快速浏览和管理命令：访问各种设置，打开插件，搜索文件，执行命令，修改文件语法等等。
⌘ + p           | 查找文件  |   快速打开你访问过的文件，模糊匹配的功能很强大，输出你能想到的任何单词就能打开目的文件。
⌘ + r           | 类/函数定义    | 列出当前文件所有的类/函数定义，输入名称就能快速移动到定义处。
⌘ + d           | 选择单词的下次出现     | ⌘ + d 会选择当前光标下单词的下一个，摁住 ⌘ 不放开，继续摁 d 会一次选择下一个，然后就能同时修改这些单词了。
⌘ + Click       | 选择任意位置    |   在所有点击的地方设置修改光标，便于同时编辑。
Ctl + ⌘ + g     |   选择所有单词  |   想要一次性编辑某个单词的所有地方，把光标放到该单词上，摁这个快捷键就能实现。
⌘ + ⇧ + Space   | 选择括号里的所有内容 | 对于编辑括号里的内容来说，非常方便。
⌘ + ⇧ + D       | 克隆一行或者所选内容 |
⌘ + \[ 或者 \]  | 增加或者减少缩进        |
⌘ + X           | 删除该行          | 把当前行删除，并放到粘贴板，可以复制到其他地方。
⌘ + /           | 注释所选行         | 根据语言自动选择注释格式
⌘ + l           | 选择该行          | 选择当前行的内容
⌘ + ⇧ + t       | 重新打开关闭的文件 |


## 3. 配置文件

配置文件是纯 JSON 格式，打开 `Sublime Text -> Preferences -> Settings - User`，来编辑个人配置， 下面是我的配置，内容都是自解释的，不需我多言：

    {
        "bold_folder_labels": true,
        "caret_style": "wide",
        "color_scheme": "Packages/Color Scheme - Default/Monokai.tmTheme",
        "file_exclude_patterns":
        [
            ".DS_Store",
            "*.pid",
            "*.pyc"
        ],
        "folder_exclude_patterns":
        [
            ".git",
            "__pycache__",
            "env",
            "env3"
        ],
        "font_size": 15.0,
        "highlight_line": true,
        "highlight_modified_tabs": true,
        "ignored_packages":
        [
        ],
        "rulers":
        [
            80
        ],
        "tab_size": 4,
        "translate_tabs_to_spaces": true,
        "trim_trailing_white_space_on_save": true,
        "vintage_start_in_command_mode": true,
        "word_wrap": true
    }


## 4. 高级设置

### 布局
在 `View > Layout` 里可以调整分屏视图，方便同时查看多个文件或者在查看代码的时候，一边在 python REPL 中测试。

### Snippets
Snippets 是提前定义的快捷键，用来生成一段内容。你可以把经常会使用到的代码块（比如文件头部的开源协议信息，自动生成 Html 的 table）定义为 snippet，然后就不用重复输入那么多的代码了。

在 `Tools > New Snippet...` 里创建新的 snippet，根据模板很容易就能完成。有什么问题就直接查看[官方文档](http://sublimetext.info/docs/en/extensibility/snippets.html)。

### 快捷键设置

在 `Sublime Text 2 > Preferences > Key Bindings - User` 里可以设置自己的快捷键。系统自定义的快捷键都在 `Key Bindings - Default`，可以仿照其中的格式来新建或者修改。

### Vintage 模式
把设置里 `"ignored_packages"` 列表里的 `"Vintage"` 删除就能开启，简单易用。VIM 使用者的福音！在 Mac 机器上，摁住某个键不放会跳出选择字符的窗口，而我们想要 Vim 的连续移动功能。只要把 Mac 的这一功能禁用：

    defaults write com.sublimetext.2 ApplePressAndHoldEnabled -bool false

如果不行的话，根据 Sublime Text 显示的名称，把上面命令的 `com.sublimetext.2` 换成 `com.sublimetext.3` 或者 `com.sublimetext`，不需要重启。

Vintage 模式并没有实现 Vim 的所有功能，主要实现的是移动功能：

    Vintage includes most basic actions: d (delete), y (copy), c (change), gu (lower case), gU (upper case), g~ (swap case), g? (rot13), < (unindent), and > (indent).

    It also includes many motions, including l, h, j, k, W, w, e, E, b, B, alt+w (move by sub-words), alt+W (move backwards by sub-words), $, ^, %, 0, G, gg, f, F, t, T, ^f, ^b, H, M, and L.

    Text objects are supported, including words, quotes, brackets and tags.

    Repeat ('.') is in there, as is specifying counts for commands and motions. Registers are supported, as are macros and bookmarks. Many other miscellaneous commands are supported too, such and *, /, n, N, s, S and more.

当然，有问题还是看[官方文档](http://www.sublimetext.com/docs/3/vintage.html)。

### 在不同机器上同步 Sublime Text 设置
在每一个机器上都重复上面的过程，无疑是一件枯燥的事情。[同步配置](https://sublime.wbond.net/docs/syncing)也是一件很简单的事情，只要把配置文件保持一致就行。亲测在 Sublime Text 2 和 Sublime Text 3 之间也有效。

## 5. python 优化

1. 打开 python 解释器：安装 SublimeREPL 插件后，输入 ⌘ + Shift + P，输入 sublimeREPL，定位到 python，摁 Enter 键打开。
2. 运行当前文件的 python 代码： ⌘ + b
4. 调试 python 代码：SublimeREPL 提供了 pdb， 使用打开 python 解释器相同的方式就能打开 pdb。
5. 安装 `Anaconda` 或者 `SublimePythonIDE` 来集成更多的 python IDE 功能



## 说明
文章会继续更新，目前只是第一个版本。

