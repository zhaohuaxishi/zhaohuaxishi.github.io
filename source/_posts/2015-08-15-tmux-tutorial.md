---
author: 郭荣飞
date: 2015-08-15
title: 利器系列之 —— 分屏利器 Tmux
tags:
 - tmux
---

**这篇文章中的 .tmux.conf 配置文件你可以在[我的代码仓库][repo]中找到**

  [repo]: https://github.com/zhaohuaxishi/dotfiles

# 为什么使用分屏工具

在 C、C++ 开发过程中分屏是一项非常提高效率的功能，因为我们时常需要一边写代码
一边编译、测试。在脱离 IDE 的时代，通常我们需要开多个终端：一个终端编辑文件、
一个终端编译运行程序、一个终端查看相关的 man 文档...。多个终端管理起来其实是
非常不便的，因为你需要不停的在各个终端之间切换。终端分屏工具是解决这个问题
的绝佳办法，它可以把一个终端分成多个窗口，把所有需要做的事情放到一个终端里面，
让你不用再耗费大量的时间在不同的窗口之间切换。

<!--more-->

# 为什么使用 Tmux

分屏工具主要分成两类，其中第一类是自带分屏功能的终端模拟器：在 Ubuntu 下面有一个
终端模拟器 Terminator 自身就支持分屏，Deepin Linux 的深度终端也支持同样的功能。

但是这些终端模拟器的分屏功能可配置性太差，你没有太多的选择。他们最大的硬伤在于
你只能使用他们的终端，所以你得忍受这些终端模拟器的蹩脚的地方，比如深度终端卡到
爆的体验。此外如果你没有安装 X11 图形界面系统，你没有办法使用这些终端模拟器。

第二类是终端分屏软件，它不依赖于终端模拟器本身，可以在任何你喜欢的终端模拟器上
运行，这一类的软件中比较有名的包括：GNU Screen 以及 Tmux。

前者的历史非常的悠久比较稳定，以 GPL 协议发布。Tmux 是 2009 年才出现的产品，
以 BSD 协议发布。很多人戏称 Tmux 为 Screen 的 BSD 版本重写，因为它包含了
Screen 大部分的功能。从个人体验来说，Tmux 相对来说比 Screen 容易使用一些
（我试过使用Screen，但是由于太笨，怎么用都不顺手）所以这里只介绍 Tmux 的使用。

# Tmux 能做什么

Tmux 的功能非常非常的强大，使用它你可以：

- 在终端模拟器中虚拟出多个窗口

- 把一个窗口分成多个区域（panel）。

- 快速在各个区域中进行复制黏贴操作。

- 配置你喜欢的快捷键。


- 使用脚本控制 Tmux。

- 实现结对编程。

- .....

本文只是简单的介绍 Tmux 基础配置以及它的分屏和复制功能，其他的功能大家可以参考
《tmux: Productive Mouse-Free Development》一书。这本书非常的薄，值得通读。

# Tmux 基本概念

为了方便理解 Tmux 的使用，这里先介绍两个相关的概念。

## 窗口（windows）

在没有 Tmux 这样的工具之前，如果我们需要在终端同时完成多项任务，比如同时听歌、
收邮件、查看文档、编辑代码，我们可能会需要开多个终端模拟器窗口来解决这个问题。
（如果连终端模拟器都没有，那估计就只能在不同的 tty 中不断的切换）

Tmux 强大之处在于，它可以让你在一个终端模拟器窗口中完成原本需要多个终端模拟器
窗口来完成的事情。它的做法是在一个终端模拟器窗口中虚拟出多个窗口，让你在不同
的虚拟窗口中完成不同的工作，并能够非常方便的在各个窗口之间进行切换。

下面是一张 Tmux 多窗口运行界面图：

![Tmux 窗口][tmux_windows]

  [tmux_windows]: /img/posts/tmux_windows.png

在上图中，我开了三个窗口，一个用来运行 VIM 编辑博客，一个用来查看 man 文档，
另一个运行 top 命令查看进程运行状况，而这些统统在 Tmux 的一个 session 中完成
（其实 Tmux 还可以运行多个 session，不过个人觉得用处不是特别大，有兴趣的可以
参考《tmux: Productive Mouse-Free Development》一书）。

## 面板

大部分情况下我们会把不同的任务分派到不同的窗口中去。但是很多时候即使是一个任务
你也没有办法在单个窗口在很好的完成它。比如 coding 这项任务，你可能需要一边写代码
，一边用 gdb 调试。又如 Git 提交这项任务，在进行 git add 提交之前我们通常是需要
知道哪些文件有改动，这个时候我通常会调用 git status 命令查看工作目录的状态，
调用 git diff 查看改动的地方。

通常我会一边参考 git status 的输出，一边查看 git diff 的输出，一边进行
git add 操作。我并不希望把这些操作分散到不同的窗口中去，因为我只是在完成一项
任务。解决这种问题可以使用 Tmux 的面板（panel），它把一个虚拟窗口分成多个可独立
操作的区域，让你在不同的区域之间协调工作完成同一项任务，如下图。

![Tmux 窗口][tmux_panel]

  [tmux_panel]: /img/posts/tmux_panel.png

上面这两个功能是我在日常开发过程中用得最多的功能，所以下面主要讲这两个功能的一些
配置。其实 GNU Screen 也同样有前面提到的这些功能，Tmux 的优势在于它的易用和容易
配置上面。

# Tmux 配置

## 准备工作

在进行真正的配置之前，你需要：

### 安装 Tmux （以 debian 系列为例）

	sudo apt-get install tmux

### 创建配置文件

	touch ~/.tmux.conf

### 去掉大写键

大写键是一个历史遗留按键，它霸占了黄金位置，却很少使用。一种比较流行的做法是把
它替换成 Ctrl 键，或者和 Ctrl 键互换。在 [EmacsWiki][] 中有一篇文章专门介绍各个
平台如何重新绑定这个键。

  [EmacsWiki]: http://www.emacswiki.org/emacs/MovingTheCtrlKey

我比较习惯的做法是直接把 `Caps Lock` 按键替换成 Ctrl。方法很简单，在你的 .bashrc
或者 .zshrc 中加入：

	setxkbmap -option ctrl:nocaps

有了这个黄金位置，许多命令输入起来会简单很多，比如输入 shell 命令时用常用的
C-r、C-a、C-e，切换 VIM buffer 常用的 C-w C-w 等等。

如果你需要输入长串的大写字符，在 VIM 中你可以先输入小写然后输入 gUw 转为
大写，在 shell 命令行中可以使用 A-u 把输入的单词变成大写。

## 绑定快捷键前缀

修改快捷键前缀是配置 Tmux 的第一步，因为 Tmux 中几乎所有的按键都是组合键，但是它
默认的前缀 C-b 输入起来有点吃力。许多人建议把它绑定到 GNU Screen 的 C-a 上去，
不过 C-a 在 shell 命令输入中是一个常用的快捷键（比如编辑一个问题突然发现没有
权限，可以通过 C-a 跳到命令行首加上 sudo 再执行），所以我把它绑定到了 C-f，因为
修改过大写键之后，这两个键输入起来是非常顺畅的。

修改前缀的方法非常简单，你只需要在 `~/.tmux.conf` 中添加：

	set -g prefix C-f
	unbind C-b

当然无论你使用哪一个前缀你都有可能和其他的软件相冲突，比如 C-a 在 shell 中使用
较多，C-f 在 VIM 中也是一个快捷键。Tmux 为了解决这个问题提供了一个功能叫做
sendkey，就是绑定另外一个快捷键，输入这个快捷键可以把 PREFIX 键发送到其他的程序。
我把这个快捷键设置成了 `PREFIX-PREFIX`。也就是说连续按两遍 C-f 可以发送一个
C-f 按键。设置方法是在 `~/.tmux.conf` 中加入：

	bind C-f send-prefix

**注意，在 Tmux 配置中 bind 后面只需要写 PREFIX 后面的部分，所以 bind C-f 其实最终
是设置了 PREFIX-C-f）**

当然这种方式并不算非常完美，因为你不得不输入两次 PREFIX，不过这个 tradeoff 我
觉得是值得的。

## 窗口（windows）的操作配置

Tmux 的窗口操作快捷键设置的是比较合理的，并不需要额外的配置，你需要做的只是记住
它的快捷键：

	PREFIX-c : 创建新的窗口（create）
	EXIT,C-d : 关闭窗口，Tmux 中的窗口和终端模拟器中的窗口一样可以通过 exit、C-d
			   来关闭。

	PREFIX-n : 切换到下一个窗口（next）
	PREFIX-p : 切换到前一个窗口（previous）

	PREFIX-w : 列出所有的窗口，以供快速切换

Tmux 创建的窗口默认是从 0 开始编号的，如果你希望从 1 开始编号可以在配置文件中
加入：

	set -g base-index 1

## 面板（panel）的操作配置

个人感觉 Tmux 本身的 panel 快捷键配置是不太合理的，所以我修改了大部分的 panel
操作快捷键。这一小节分三个部分讲解 Tmux 的面板操作 —— 也就是传说中的分屏操作。

### 分屏

Tmux 支持水平分屏和垂直分屏，但是 Tmux 默认的快捷键很难输入。默认垂直分屏是
`C-"` 而默认水平分屏是 `C-%`。我习惯的做法是使用最直观的 `-` 表示水平分屏，
`|` 表示垂直分屏，不过因为 `|` 和 <code>\</code> 一般在同一个按键上而后者不需要按住
`SHIFT` 来转换，所以我一般直接把 <code>\</code> 绑定为垂直分屏。

在你的 `~/.tmux.conf` 文件中加入：

	unbind '"'
	bind - splitw -v

	unbind %
	bind \ splitw -h

现在你可以使用 `PREFIX--`（按下 PREFIX 组合键之后按下 - 键）来进行水平分屏，
使用 <code>PREFIX-\</code> 来进行垂直分屏。

和窗口一样，Tmux 分出来的 panel 默认是从 0 开始编号，你可以通过在配置文件中
加入：

	set -g pane-base-index 1

使编号从 1 开始。

### 切换面板

分屏之后最重要的一个操作应该是在不同的 panel 之间切换，我是一个 VIM 重度患者，
所以习惯把涉及到方位操作的所有按键绑定到 hjkl 上面去。

我的配置如下：

	bind-key h select-pane -L
	bind-key j select-pane -D
	bind-key k select-pane -U
	bind-key l select-pane -R

使用 PREFIX-[hjkl] 就可以轻松的切换到不同方向的小窗口中去。

Tmux 还可以通过 `PREFIX-q` 在所有的面板上显示一个编号，输入这个编号可以直接
跳转到这个这个面板上去。

### 修改当前面板的大小

Tmux 的默认分屏是对半分，不过更多的时候我们需要的是一个较大的主窗口和几个
小一点辅助窗口。比如一个大的 VIM 窗口编辑源代码，一个小的窗口用来编译源码。
修改完源码之后可以立即切换到小窗口进行编译是一件很幸福的事情。Tmux 提供了
改变 panel 大小的功能，我依旧把它绑定到 hjkl 四个按键上，只不过这一次使用的是
C-[hjkl]，也就是说最终可以通过 PREFIX-C-[hjkl] 来改变当前窗口的大小，配置如下

	bind -r ^k resizep -U 5
	bind -r ^j resizep -D 5
	bind -r ^h resizep -L 5
	bind -r ^l resizep -R 5

如果你是两个垂直并列的 panel，可以使用 PREFIX-C-[hl] 来调整它们的大小，
如果你是两个水平并列的 panel，可以使用 PREFIX-C-[jk] 来调整它们的大小。
你可以通过把上面的 5 改成你想要的值来调整单次 resize 的粒度。

此外如果你想要暂时把其中一个 panel 最大化，你可以使用 `PREFIX-z`

Tmux 还为 panel 提供了几种默认的布局，你可以通过 `PREFIX-space` 来切换这些
布局。

## 复制黏贴的操作

前面几个小节系统的介绍了窗口和面板的使用，下面重点介绍一些 Tmux 的拷贝模式
（copy-mode）。

我们经常需要在一个窗口（或者 panel）中复制代码黏贴到另一个窗口（或者 panel）中
（虽然这不是一个很好的习惯）。Tmux 提供了强大的的复制黏贴功能来帮助我们完成
这一任务。

在 Tmux 中复制是在拷贝模式下完成（copy-mode），进入拷贝模式的默认快捷键是
`PREFIX-[`，不过我习惯把它绑定到 `PREFIX-C-v` 上面去

	bind ^v copy-mode

进入了拷贝模式之后可以使用 hjkl 移动光标到你想要拷贝的地方。找到需要复制的部分
之后按空格键开始复制，使用 hjkl 选择复制的区域，最终按回车完成复制退出拷贝模式。
同样这些按键，你也可以自己绑定快捷键。我习惯了 VIM 的复制黏贴，所以我把 v 绑定
为拷贝模式下复制的开始（默认是 SPACE），y 绑定为复制结束（默认是 ENTER）。

	bind -t vi-copy v begin-selection
	bind -t vi-copy y copy-selection

复制完成之后你可以切换到你想要的黏贴的窗口，输入 `PREFIX-]` 完成黏贴操作，当然
这个按键你同样可以绑定，比如我把它绑定到 `PREFIX-C-p` 上面。

	bind ^p pasteb

# Tmux 使用中的其他问题

## 去掉鼠标

Tmux 的一个口号是 mouse free，有了它之后你可以抛弃鼠标，如果你想要禁用鼠标
设置，你可以在你的 `~/.tmux.conf` 文件中加入：

	set -g mode-mouse off

## 颜色问题

Tmux 在很多终端下使用的时候会出现颜色错乱的问题，为了解决这个问题你需要设置
TERM 环境变量，最简单的做法是给 tmux 起一个别名，如下：

	alias tmux='TERM=xterm-256color tmux'

把上面这句写到你的 `.bashrc` 或者 `.zshrc` 中即可。

****

Tmux 网上有很多相关的资料可以参考，最全面的资料应该是
《tmux: Productive Mouse-Free Development》一书，强烈推荐有兴趣了解 Tmux 的
人看一看这本书。聪明的你一定很容易找到这本书。
