---

layout: post
title: "使用 vi 模式操作tmux 屏幕"
excerpt: "抛弃鼠标，拥抱键盘。现在可以直接选择屏幕上的文字啦，tmux 和 vim 果然神器。"
categories: blog
tags: [vim, mac, tmux]
comments: true
share: true
---

使用 tmux，配合上 vim，基本上可以告别鼠标。之所以说基本上，是因为要复制 tmux 屏幕上的内容一直还是用鼠标选择，然后 `Cmd+C`，`Cmd+V`。

这篇文章就讲讲 Mac 系统上，怎么像 VIM 那样在 tmux 屏幕上移动和选择文字。

## 配置

### 使用 vim 模式
Tmux 支持 vim 模式，这样就可以使用 `hjkl` 来上下左右移动，`w`跳一个单词等等。只要在 `~/.tmux.conf` 文件加上下面这句配置：

    setw -g mode-keys vi

### 把内容自动复制到系统粘贴板
默认情况下，在 tmux 上选择的文字，只能在 tmux session 使用。要想在所有地方都可以粘贴，就要把文字内容复制到系统的粘贴板：

1. 安装 retach-to-user-namespace： `brew install reattach-to-user-namespace`
2. 更新 `~/.tmux.conf` 配置文件：`set-option -g default-command "reattach-to-user-namespace -l bash"`

### 用 `v` 和 `y` 来选择和复制
要想在选择文字之后，像 vim 那么复制粘贴，要这么配置：

    bind-key -t vi-copy v begin-selection
    bind-key -t vi-copy y copy-pipe "reattach-to-user-namespace pbcopy"


## 使用
设置好之后呢，使用起来就比较简单：

1. 摁 `ctrl-b [` 进入 vi 模式
2. 使用 `hjkl`，`wb` 等 vi 快捷键移动到目标位置
3. 摁 `v` 键，进入 Visual 模式，开始选择内容
4. 继续使用 vi 快捷键移动，直到要复制的内容被选中
5. 摁 `y` 键复制内容到粘贴板
6. `Cmd-V` 把内容放到想要的地方