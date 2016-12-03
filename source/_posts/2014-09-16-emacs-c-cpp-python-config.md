---
author: 郭荣飞
date: 2014-09-16
title: Emacs C、C++、Python 编程的简单配置
tags:
 - Emacs
---

# 简介

Emacs 和 Vim 并称 Linux 下的两大编辑神器（不是 IDE，只是编辑器）。一般人认为这两
个编辑器的门槛太高，不容易学，使得很多人都只能敬仰却没有勇气使用它们。其实个人并
不认同这样的看法，Emacs 和 Vim 基本的操作其实并不复杂，而且它们各自都带有非常好
的入门教程。Emacs 在启动页面上会有一个非常通俗易懂的教程（tutorial），如果你关闭
的启动页面，你可以在菜单栏的 Help 下找到这个教程，如果你关闭的菜单栏可以使用 M-x
（也就是 Alt + x）执行命令 menu-bar-mode 显示菜单栏。个人觉得这份教程对于初学者
的价值比任何其他资料都更高，如果你没有使用过 Emacs 或者说你还不是很熟悉它的话，
建议你先阅读这份文档。

<!--more-->

Emacs 和 Vim 都有非常高的可定制性，你可以通过配置文件和插件对它们进行高度的定制
，使得这个编辑器使用起来非常的称手（有它该有的功能，最主要是不该有的功能它没有）
。这大概也是它们比一般的 IDE 更受欢迎的主要原因吧。

当然凡事都有它的利弊，好的东西总是有点代价，那就是配置其实还是挺花时间的。我使用
Emacs 的时间其实不是很长，一路走来可以说是跌跌撞撞，目前也只能算是相对熟悉这个编
辑器而已。写这篇文章不是为了炫耀自己的配置有多厉害，而是希望能够总结自己的一些经
验，分享给大家，以减轻初学者在配置 Emacs 时的痛苦。本文如果有疏漏、错误的地方欢
迎大家指正出来。

# 配置目标

由于本人主要使用 Emacs 做 c、c++ 的开发，所以本文的配置目标主要是把 Emacs 配置成
一个适合 c 和 c++ 的开发工具。另外我也在闲暇之余学一点 Python，所以这里的配置也
会涉及到 Python 的相关配置。

不过就编程而言，其中有很大一部分的配置对于各种语言的开发都是一样的，涉及到具体语
言的配置其实并不多。所以如果你想要 Emacs 做其他项目的开发，本文中的配置可能仍然
对你会有一些帮助。

本文的配置是基于 Emacs 24 进行的，如果你目前安装的版本低于这个版本建议你升级到该
版本。因为 Emacs 24 自带了包管理器 package，有了它安装和配置插件的难度可以大大的
降低。

# 配置文件及其结构

在讲解具体的配置之前，我想先说一下配置文件和它的格式。所谓的配置文件官方名称叫做
初始化文件（initialization file，或者 init file），这个文件如果存在的话，Emacs
在启动的时候会加载并解析它。Emacs 会查找以下三个文件： ‘~/.emacs’, ‘~/.emacs.el’
, 或者‘~/.emacs.d/init.el’，其中 ’～‘ 表示你的 home 目录。所以你可以使用这三者之
一作为你的配置文件，相对来说第一个用的多一些。

结构主要是说你的配置文件按照怎么样一种布局书写，Emacs 本身并不要求你的配置文件按
照一定的结构来组织，但是一个良好的配置文件结构可以使得你的配置一目了然，同时也非
常方便你日后的查阅和修改（你一定会经常改动它，因为你会爱上 Emacs 的配置）。这就
像是任何一种语言都会有的编程规范，编译器不会要求你使用何种规范，但是好的规范可以
让人非常容易的读懂代码。记住编程首先是写给人看，然后才是写给机器看（这句话出自《
代码大全》）。

我按照下面的方式组织我的配置文件，你可以不用这种方式，但是你需要有自己固定的方式
。

  1. 文档注释
  2. 插件无关的公共设置（Emacs 自带的一些设置）
  3. 公共插件配置（和语言无关的插件配置）
  4. 特殊配置（如针对单个语言的具体插件配置）
  5. 按键绑定（把它放在最后面是因为按键绑定可能会涉及到相关插件）

# 配置内容

下面我们开始说具体的配置内容，我们将会按照上面提到的结构逐点展开。完整的配置文件
可以从 [这里][emacsgithub]下载.

 [emacsgithub]: https://github.com/zhaohuaxishi/emacsbackup

## 插件无关的公共设置

### 在 mode line 上显示行列号

首先要说明一下什么是 mode line，一个刚刚启动的 Emacs 界面可以大致分成一下几个区
域。最上面是菜单栏，下面紧挨着工具栏（为了得到较大的编辑区域，上面这两块一般会被
隐藏起来）。在最底部是一个 echo 区域（echo area，才疏学浅实在不知道如何翻译 echo
为好），这个区域用来显示输出信息，同时也用来输入信息，比如执行 Emacs 命令。夹在
中间的大块区域在 Emacs 中被称为窗口（window），这里的窗口和图形界面里面的窗口不
一样，在 Emacs 中图形界面的窗口一般称为框架（frame）。我们所谓的 mode line 就是
一个窗口最下面突起的那一行。

把以下语句添加到配置文件中可以在 mode line 中显示行列号

```lisp
    ;; 在 mode line 上显示行列号
    (setq column-number-mode 1)
    (setq line-number-mode 1)
```

你也可以使用 linum-mode 在 buffer 的左边列出没一行的行号。你可以通过 M-x
linum-mode 来暂时打开和关闭行号显示，也可以把 (global-linum-mode) 加入到你的配置
文件中，默认的显示行号。

### 去掉滚动条、菜单栏和工具栏

大部分时候我们希望能够得到较大的编辑区域，Emacs 提供了相关的配置使得你可以最大化
你的编辑区域，你可以通过以下代码（写到配置文件里面）来去掉滚动条、菜单栏、工具栏
。

```cpp
    ;; 取消滚动栏，工具栏
    (tool-bar-mode 0)
    (menu-bar-mode 0)
    (scroll-bar-mode 0)
```

这个配置一般会有两种形式，你也许还会看到下面这种形式（tool-bar-mode nil）。在
Emacs 24 中使用 nil 好像不能达到上面的效果，但是其他的一些配置使用 nil 又可以，
我不知道是什么原因，但为了统一风格，我一律使用 0 表示否定值。

### 关闭 Emacs 的起始欢迎页面

Emacs 的起始欢迎页面对于初学者来说帮助比较大，可以了解各方面的信息，包括官方的教
程。但是对大部分人来说它有点碍事，也影响美观。下面的语句可以关闭它。

```cpp
    ;; 关闭emacs启动时的页面
    (setq inhibit-startup-message 1)
    (setq gnus-inhibit-startup-message 1)
```

### 以 y/n代表 yes/no

Emacs 经常需要我们确认一些信息，比如你在没有保存文件的情况下关闭 Emacs，它会要你
确认是否保存文件，这个时候你需要输入 yes 来表示确定。这很繁琐，你可以使用下面的
语句改变这种交互方式，把 yes、no 改成 y、n，这样可以很方便的输入。

```cpp
    ;; 以 y/n代表 yes/no
    (fset 'yes-or-no-p 'y-or-n-p)
```

### 备份设置

使用 Emacs 编辑文件，经常会发现多出一些以 ~ 结尾的文件，如果你是完美主义者或者你
有轻微强迫症（我就是），这是很难忍受的，我原来经常会去删除这些文件，直到后来找到
了更好的解决方式。在 Emacs 中你可以设置你自己的备份策略，包括备份几份数据，备份
到哪里等等。下面是我的备份策略设置：

```cpp
    ;;; 设置备份策略
    (setq make-backup-files t)
    (setq kept-old-versions 2)
    (setq kept-new-versions 2)
    (setq delete-old-versions t)
    (setq backup-directory-alist '(("" . "~/.emacsbackup")))
```

如果你完全不想备份可以直接把 make-backup-files 设置为 0，不过备份文件在很多时候
还是非常有用的，所以我并不建议你那样做。如果你不想备份文件出现在你的工作目录下，
你可以设置一个目录专门存放你的备份文件，然后使用 backup-directory-alist 这个变量
设置你的目录，比如我就把它设置在自己的 home 目录下的一个隐藏文件夹（.emacsbackup
）下面。

以上配置中间的三条语句是指明你的备份策略，保存最老的 2 份数据和最新的 2 份数据，
删除其余的备份。你可以自己改变其中的值，以保存你想要的份数。

## 公共插件配置

Emacs 的插件数不胜数，你想要的各种功能都可以通过相关的插件来完成，这里介绍一些我
平常使用较多的插件。主要包括一下这些插件：

  - package: 包管理插件
  - autopair: 括号的自动匹配
  - yasnippet: 代码块扩展
  - auto-complete: 代码智能补全
  - evil: 给 emacs 加入 vim 的功能
  - desktop: 保存工作环境
  - ido: 快速的查找文件、切换 buffer
  - smex: 快速的输入 emacs 命令

  - gtags: 代码跳转，源码阅读神器
  - fill-column-indicator: 用于提示每行的代码字符限制
  - doxymacs: 写注释非常方便

  - jedi: 唯一需要的 python 插件。功能应有尽有

其中第一部分的插件是一些公用的插件，和具体的编程语言无关，第二部分的插件我主要用
在 c、cpp 开发中，第三部分的插件主要用在 Python 开发中。这里主要介绍第一部分的插
件。

### 包管理插件 —— package

第一个要隆重介绍的插件是 package，这个插件是 Emacs 24 自带的插件，主要用来方便的
管理各种插件。介绍它之前需要先介绍一下 ELPA 这个概念，ELPA 是 Emacs Lisp Package
Archive 的缩写，它相当于是一个插件（也就是 Lisp Package）仓库，里面有各种各样的
打包好了的 Emacs Lisp 代码（插件用 Emacs Lisp 语言编写，Emacs Lisp 是 Lisp 语言
的变种，级别应该相当于是 Scheme 语言，我们编写的配置文档其实也是 Emacs Lisp 语言
编写）。

我们知道 Emacs 的可定制性和扩展性都非常的高，这也就使得 Emacs 的插件不计其数，安
装和管理这些插件变得非常的繁杂。为了解决这个问题，就出现了 ELPA，把插件统一起来
。但是它只是一个仓库，我们需要一种便捷方式从仓库中下载和更新我们的插件。而这种便
捷的操作依然是通过插件完成，也就是这里提到的 package 插件，它内置在 Emacs 24 中
。如果你使用的不是 Emacs 24，建议你先升级到 24 版本。

因为 package 是一个内置的插件，所以你不需要安装，要使用它你只需要告诉它仓库（
ELPA）的位置即可。ELPA 有比较多，目前大家使用较多的是 MELPA，里面的插件非常丰富
，我们这里提到的插件都可以在上面找到。下面是 package 的设置：

```cpp
    ;; 设置 package system，这里使用 MELPA，里面的包可以说是应有尽有
    (require 'package)
    (add-to-list 'package-archives
                 '("melpa" . "http://melpa.milkbox.net/packages/") t)
    (package-initialize)
```

注意不要漏掉了最后的 (package-initialize)。设置好了之后，我们可以通过执行
list-packages  命令（通过 M-x 然后输入指令）来查看所有的包，在查找到你想要的包之
后在改行输入 i 标记你要安装这个包，然后输入 x 执行你的安装指令，这样这个插件就会
自动安装到你的 Emacs 中去了（在 ~/.emacs.d/elpa/ 目录下面）。如果有包需要更新，
在你执行 list-package 命令之后在 echo 区域会有提示，此时直接输入 U 然后输入 x 可
以更新这些包。当然你也可以很方便的删除你安装的包，方式是在你想要删除的包上面输入
d 然后输入 x 执行它。

很方便吧，后面提到的插件都是通过这种方式来进行安装，以后不再重复。

### 括号自动匹配 —— [autopair](https://github.com/capitaomorte/autopair)

在编写大部分程序的时候我们都需要括号匹配，所以很多适合我们输入 '(' 的适合非常希
望能 Emacs 自动补全 ')' 并且把光标停到两者中间。可是 Emacs 并默认没有提供这个功
能，它包含一个自带的模式 electric-pair-mode 可以完成这个工作，你可以使用 M-x 来
开启它也可以把它加入到你的配置文件中去。我尝试过使用 electric-pair-mode 但是感觉
不如 autopair 好用，最后还是选择了 autopair。

通过 package 安装之后你可以通过 M-x autopair-mode 来开启这个模式，也可以把他加入
到你的配置文件中默认开启它，或者针对某种主模式开启。对于 Emacs 的大多是插件，都
可以有这三种方式来使用它：

- 通过命令行，在需要的时候使用，比如:

    M-x autopaire-mode

- 通过配置文件，默认使用，比如把以下语句写入到配置文件中：

```cpp
    (autopair-global-mode)
```

- 通过配置文件，对特殊的主模式下使用，比如：

```cpp
    (add-hook 'c-mode-common-hook 'autopair-mode)
```

上面提到的三种方式对于大部分的插件都是通用的，不过不同的代码还是会有不同的使用方
式，重复它们其实没有太大的意义，最佳的方式是 google 一下，查找到这个插件的主页（
一般在 github 上），然后通过看上面的官方使用文档确定如何进行下一步的配置。我在每
个插件的名字上加了官网链接，方便大家参考查阅。

### 代码扩展神器 [yasnippet][yas] 和代码补全神器 [auto-complete][acp]

  [yas]: https://github.com/capitaomorte/yasnippet
  [acp]: http://cx4a.org/software/auto-complete/manual.html

这两款插件可以说是编程必备的。其中 yasnippet 是用来做代码扩展的，比如你在编写 .c
文件时输入 io 然后按下扩展键，它会自动扩展成 #include <stdio.h>，输入 d 然后按下
扩展键之后会自动补全成 #define，非常的便捷。而 auto-complete 则是用来做代码补全
的，比如你在 .c 文件中定义了一个函数 `my_function`、和一个变量 `my_name`，在其他
位置输入 my 的时候它会把 `my_function` 和 `my_name` 列出来供你选择，这个功能基本
上的 IDE 都有提供，相信大家并不陌生。

这两款插件可以说殊途同归，虽然它们使用的原理不一样，但是它们最后都大大提高了代码
输入的速度。前者主要通过预先定义好用于扩展的 snippet，而后者好像主要是通过对
buffer 中的数据进行分词来提供补全的来源。

不过这两款插件同时使用了 TAB 键， yasnippt 用它来作为扩展键，而 auto-complete 用
来是作为提示触发键和选择键。为了解决这个冲突，我试过两种方法：第一，关闭
auto-complete 的自动提示，这样当你按下 TAB 的时候如果有相关的 yasnippt 代码扩展
，会先进行 yasnippt 扩展，然后是 auto-complete 补全提示。第二种方式就是更改其中
之一的促发按键，我目前使用的就是这样中方法。下面是我的相关设置:

```cpp
    ;; yasnippet setting
    (yas-global-mode 1)
    (define-key yas-minor-mode-map (kbd "<tab>") nil)
    (define-key yas-minor-mode-map (kbd "TAB") nil)

    ;; 设置 f2 为 yas 扩展快捷键
    (define-key yas-minor-mode-map (kbd "<f2>") 'yas-expand)

    ;; auto-complete
    (ac-config-default)
```

这里主要是把 yasnippt 的扩展键去掉，然后把他重新绑定到 f2 上（按键绑定的设置我一
般放在了配置文件的最末尾，写在这里是为了方便讲解）。

### Vim 和 Emacs 的融合 —— [EVIL][evil]

  [evil]: https://gitorious.org/evil/pages/Home

Evil，可以说插件正如其名，功能强大到邪恶的地步。在 Linux 领域有一场旷日持久的编
辑之战，使用号称“编辑器之神”的 Vim 的 V 党和使用号称“神的编辑器”的 Emacs 的 E 党
吵的不可开交，只为了争夺最好用的编辑器的虚名。我是一个实用主义者，我虽然使用
Emacs，但是我承认 Vim 一样非常强大，它的 normal mode 用来查看文档要比 Emacs 好用
的多，Vim 可以通过 hjkl 来移动光标，而 Emacs 需要按 C-p C-n C-f C-b 这些组合键。
但是 Vim 的编辑能力确实相对没有 Emacs 强， Vim 在输入的时候无法使用 hjkl 来定位
光标，而 Emacs 的按键仍然可用。

因此最好的方式就是把两者的精华融合在一起，得到世上最强大的编辑器 “Emacsim”，这也
就是这里要介绍的 Evil。**E**xtensible **Vi** **L**ayer for Emacs，它是 Emacs 的
一个 Vi 扩展层，使得我们在 Emacs 上可以使用 Vim 的强大功能，比如它的选择功能和方
便的文本阅读功能等等。

不过这两大神器的融合难免有一些不和谐的地方，比如按键上的冲突是在所难免的，好在
Evil 提供了重置按键的功能。下面是我收集的一些按键重绑定。这些按键的重新绑定可以
使得两者的融合更加的融洽，你可以根据自己的需求进行按键从绑定。

```cpp
    ;; evil 把 emcas 和 vim 的精华糅合在一起
    (evil-mode 1)

    ;; normal mode 下可以使用 ",," 来切换 buffer，取消 q 按键绑定
    (define-key evil-normal-state-map (kbd ",,") 'evil-buffer)
    (define-key evil-normal-state-map (kbd "q") nil)

    ;; 输入模式下取消下面这些按键绑定，使得在输入的时候可以高效的移动光标
    (define-key evil-insert-state-map (kbd "C-e") nil)
    (define-key evil-insert-state-map (kbd "C-d") nil)
    (define-key evil-insert-state-map (kbd "C-k") nil)
    (define-key evil-insert-state-map (kbd "C-g") 'evil-normal-state)

    ;; 其他绑定
    (define-key evil-motion-state-map (kbd "C-e") nil)
    (define-key evil-visual-state-map (kbd "C-c") 'evil-exit-visual-state)
```

因为 Vim 使用的是多模式编辑，Evil 也自然是多模式的，在不同的模式之间切换的时候，
默认会在 Emacs 的 mode line 上标记目前的模式，不过默认的方式太不显眼不是很容易识
别。我们希望 Emacs 可以很显眼的提示让我们目前是什么模式，Emacs 提供了设置每个模
式颜色的功能。下面这段代码来自 Emacs wiki，它可以使得不同的模式有不同的颜色显示
：

```cpp
    ;; change mode-line color by evil state
    (lexical-let ((default-color (cons (face-background 'mode-line)
                                       (face-foreground 'mode-line))))
      (add-hook 'post-command-hook
                (lambda ()
                  (let ((color (cond ((minibufferp) default-color)
                                     ((evil-insert-state-p) '("#e80000" . "#ffffff"))
                                     ((evil-emacs-state-p)  '("#444488" . "#ffffff"))
                                     ((buffer-modified-p)   '("#006fa0" . "#ffffff"))
                                     (t default-color))))
                    (set-face-background 'mode-line (car color))
                    (set-face-foreground 'mode-line (cdr color))))))
```

这种方式在终端开启 Emacs（emacs -nw）的效果非常的好，但是使用 x-window 的话效果
不佳，如果你基本上都使用 x-window 启动 Emacs，那么你可以考虑使用 powerline 插件
。它原本是 Vim 的一个插件，目前有 Emacs 的版本，当然随着 Evil 的到来自然会有
powerline-evil 插件，安装 powerline 的时候会自动安装 powerline-evil。使用这个插
件之后 Evil 下的不同模式显示就近乎完美了。


### 保存工作环境 desktop 和 session


每次关闭 Emacs 之后我们打开的 buffer 都会随之关闭，很多时候我们需要保存我们的工
作环境，下次打开 Emacs 的时候希望原来打开的 buffer 依然存在。这个时候我们可以借
助 desktop 这个插件，它是一个内置插件，不需要安装只需要打开相应的模式即可，在关
闭 Emacs 的时候 desktop 会保存你开启的 buffer，这样下次打开工作环境还在，就好像
没有关闭过一样。

另一个类似的插件是 session，它可以 File 菜单下增加子菜单，记录我们最近访问的的文
件，就像大部分 IDE 提供的功能一样。不过因为我大部分时间关闭 menu-bar 而且使用了
desktop，这个插件的作用对我来说几乎为零，很少使用。


### 健步如飞的输入 —— ido、smex


这两个插件也是重量级的，我非常喜欢。Emacs 的输入大部分时候都可以通过 TAB 键来补
全，但是没有一个直观的界面显示，这两个插件相当于是输入指令时的 auto-complete，它
可以大大减少输入的工作量。

ido 可以用来快速的查看文件和切换 buffer，当我们使用 ido 来打开文件的时候，我们可
以看到一个递进的文件层次结构，起初显示最顶层的目录和文件，当我们进入一个文件夹之
后显示该目录下的子项目。同样在切换 buffer 的时候，有了 ido 可以直观的看到有哪些
buffer，在你输入想要切换的 buffer 名称的时候 ido 也同时帮你进行筛选。此外 ido 本
身有记忆功能，可以让你快速的定位到上一次打开的文件夹。

ido 使得查找文件和切换 buffer 的任务变得异常的轻松，smex 则使得输入 Emacs 命令变
得非常便捷。smex 提供一个非常类似于 ido 的界面。在你输入命令的时候直观的给出后选
项，在输入的同时不断的筛选，同时也记录你最近使用过的命令，使得你可以很方便的输入
命令（其实大于大多数人来说，常用的命令也就那么几条，有了 smex，基本上不用输入多
少个字符就可以找到你想要的命令，因为它有记忆功能）。


### 纯文本编辑利器 —— org


org mode 的强大并不是三言两语可以说清楚，它的官方文档就有好几百页。我本人并不是
很熟练的掌握它这里也不做过多的介绍，有兴趣的可以去看看它的官方网站，上面有官方文
档。

org 主要可以用来编写文档，它可以提供层次化的结构，使得你写作的时候可以条理清晰，
此外 org mode 实现了 GTD （好像是 Get Things Done 的缩写），它可以让你列出自己的
计划，给出你的完成时间，计划时间，提供一份日程表，让你对自己的生活进行很好的规划
。它还可以很方便的插入表格，转换成各种文件格式。也有专门用来写博客的 org2blog 插
件让你可以通过 Emacs 的 org 模式编写博客。

我对 org 并不是很熟，在这里只是简单的介绍和推荐，如果你想要更升入的了解它的功能
可以去看它的官方文档。


### 其他插件


其他还有一些插件也很给力，比如 auto-complete-clang 可以在 auto-complete 的基础上
，通过 clang 作为后端提供库函数的补全。ECB（Emacs Code Browser），可以让你像使用
IDE 一样使用 Emacs，它提供了文件的各种视图。你可以查看项目的目录结构、目录下的源
文件、源文件中的函数列表等等。

这些插件虽然很强大，但是会让 Emacs 变的很慢，一定意义上来说反而降低了 Emacs 的效
率，所以这里不做过多的介绍，如果你有兴趣，可以去它们的官网上查看具体的使用方式。


## 单个语言的特殊配置和GUI的特殊配置


前面的配置都是一些通用的配置，并不限于某个语言。下面的配置则更多的是对于单个语言
的特殊配置。


### c、cpp 插件配置


#### 行字符数限制 —— [fill-colum-indicator][ficw]

  [ficw]: https://github.com/alpaker/Fill-Column-Indicator

c、cpp 的一个编程规范就是每一行的字符不要超过 80 个字节，而我们在编程的时候不可
能去记住没一行目前有多少字节，我们需要一个直观的提示，这也就是
fill-colum-indicator 的作用。它会在屏幕的右边规定个字节的地方给出一条竖线，使得
你可以很直观的看到自己的代码是否超过了这个界限。你可以设置它指示的位置，颜色和竖
线的宽度。

```cpp
    (setq fci-rule-column 80) ;; 80 个字节处画竖线
    (setq fci-rule-color "orange") ;; 竖线为黄色
    (setq fci-rule-width 2) ;; 竖线宽度为 2 个像素
    (fci-mode 1) ;; 开启 fci 模式
```

#### 代码跳转 —— [gtags][gtagsw]

  [gtagsw]: http://www.gnu.org/software/global/globaldoc_toc.html#Emacs-editor

在编写或者阅读较大型的项目的时候，我们通常希望可以在函数的定义和实现中进行跳转，
这个功能大部分 IDE 都有提供Emacs 也可以实现它。Emacs 中实现代码跳转的方式很多，
比如 etags、cscope 等等，我个人比较习惯的使用的是gtags，它以 GNU GLOBAL 为依托，
所以在使用 gtags 之前你必须先安装 global。在官网的的源码包里有 gtags.el 这个插件
（在 MELPA 中有一个 ggtags 是 gtags 插件的扩展，个人觉得还不如这个原生的好用），
如果你编译安装 global，那么在 /usr/local/share/ 目录下面会有 gtags.el 这个文件，
你可以把这个目录加到你的 load-path 中去，也可以自己建立一个文件夹（比如：
~/.emacs.d/plugins/），把源码包中的 gtags.el 放到该文件夹下之后把你建立的文件夹
添加到 load-path 中。

```cpp
    (setq load-path (cons "~/.emacs.d/plugins/" load-path))
    (autoload 'gtags-mode "gtags" "" t)
```

目前我用它来阅读 Linux Kernel 的代码，个人感觉还是非常好用的，所以推荐给大家。


#### 方便而美观的代码注释 —— [doxymacs][doxymacsw]

  [doxymacsw]: http://doxymacs.sourceforge.net/

Emacs 本身的注释功能已经非常强大了，比如使用 M-; 可以插入注释，在注释里面使用
M-j 可以形成多行注释。如果你想要一个更加强大的注释插件，那么可以使用 doxymacs，
它可以用来插入整个文档的注释，函数的的注释，多行注释，单行注释等等。它还可以进行
注释的语法高亮，非常的好用。

不过 doxymacs 也需要编译安装，在 package 里面没办法直接安装这个插件。编译安装
doxymacs 相对来说复杂一些。具体的方式可以参考官方文档的。


#### 自定义函数


为了让上面这些配置只对 c、cpp 有效，我们可以定义一个函数，在函数里面写入我们的配
置，然后把这个函数挂到 c 和 cpp 的 mode 上面：

```cpp
    (defun my-c-mode-hook()
    "This is the function of c mode hook"

    ;; set fci-mode argument
    (setq fci-rule-column 80)
    (setq fci-rule-color "orange")
    (setq fci-rule-width 2)
    (fci-mode 1)

    ;; gtags mode and doxymacs
    (setq load-path (cons "~/.emacs.d/plugins/" load-path))
    (autoload 'gtags-mode "gtags" "" t)

    ;; open gtags mode
    (gtags-mode 1)

    ;; doxymacs mode
    (doxymacs-mode 1)

    ;; 回车换行自动缩进
    (setq-default indent-tabs-mode nil)
    (global-set-key (kbd "RET") 'newline-and-indent)

    ;; 缩进风格设置
    (setq c-default-style '((java-mode . "java")
                                         (awk-mode . "awk")
                                         (other . "linux")))
    )
```


函数定义好了之后，通过 hook 把它挂到 c-common-hook 上面。

```cpp
    (add-hook 'c-mode-common-hook 'my-c-mode-hook)
```


### Python 插件配置

Python 的配置除了前面提到的 fill-colum-indicator 之外（Python 一般要求每行不超过
79 个字符，而 c、cpp 则一般是 80 个字符，所以虽然这个插件通用，我还是分开写它们
的配置）我只推荐一个插件——[jedi][jediw]。这个插件功能非常的强大，它可以用来实现
代码补全，代码跳转， docstring 的查看等等。

  [jediw]: http://tkf.github.io/emacs-jedi/latest/

jdei 的安装相对要麻烦一下，主要分为一下几个步骤：

- 安装相关依赖，包括 virtual、epc、jedi

```cpp
    sudo pip install virtualenv
    sudo pip install epc
    sudo pip install jedi
```

- 用 package 安装 jdei 插件(package 在安装这个插件的同时会安装它依赖的的插件)

- 在 Emacs 中执行命令 jedi:install-server


安装完成之后可以通过以下配置开启 jedi:

```cpp
    ;; setup jedi
    (jedi:setup)
    (setq jedi:complete-on-dot t)
```

同样的，你可以定义一个方法把上面的配置写到函数中，然后通过 hook 挂到 python mode
中。

jedi 的具体使用方式，请查考它的[官方文档][jediw] 下面是一些基本的快捷键：

```cpp
    ;; <C-tab> jedi:complete
    ;; Complete code at point.

    ;; C-c ? jedi:show-doc
    ;; Show the documentation of the object at point.

    ;; C-c . jedi:goto-definition
    ;; Goto the definition of the object at point.

    ;; C-c , jedi:goto-definition-pop-marker
    ;; Goto the last point where jedi:goto-definition was called.
```

jedi 在 python 3 下好像并不能很好的工作，建议使用 Python 2 来配置它。（我在 Arch
下配置的时候因为它以 Python3 作为默认的 Python 版本配置没有成功，把默认改成
Python2 之后可以正常工作）。


### GUI 特殊配置


Emacs 中还有一些配置只有在使用 x-window 的时候才有效，比如设置字体，使用
powerline 和系统共享剪贴板等等。

为此我专门为 x-window 配置定义了一个函数，当使用 x-window 的时候执行该函数，以配
置 GUI 特殊功能。

```cpp
    (defun x-window-config()
        (powerline-center-evil-theme)
        (setq x-select-enable-clipboard 1)
        (set-default-font "Courier 10 Pitch"))

    (if window-system (x-window-config))
```


## 按键绑定


按键绑定之所以放在最后面是因为按键的绑定可能涉及到前面提到的插件，在所有的插件都
配置完成之后设置按键绑定可以清晰明了，也防止冲突，这里的按键绑定主要是 f1-f12 的
绑定，具体绑定如下，你也可以绑定其他的按键：

```cpp
    ;; 设置 f1 打开帮助文档
    (global-set-key [f1] 'info)

    ;; 设置 f2 为 yas 扩展快捷键
    (define-key yas-minor-mode-map (kbd "<f2>") 'yas-expand)

    ;; 设置 f3 关闭当前buffer
    (global-set-key [f3] 'kill-this-buffer)

    ;; 设置 f4 打开 eshell
    (global-set-key [f4] 'eshell)

    ;; 设置 f10 为查找符号的引用
    (global-set-key [f10] 'gtags-find-rtag)

    ;; 设置 f11 为查找符号的定义
    (global-set-key [f11] 'gtags-find-tag)

    ;; 设置 f12 返回原来的位置
    (global-set-key [f12] 'gtags-pop-stack)
```

其中 f1 是默认的绑定，f2 作为 yasnippt 的扩展键，f3 用来关闭当前 buffer（Emacs
的C-x k 是默认的关闭 buffer 按键）。f10 到 f12 是专门绑定到 gtags 插件的相关函数
上的，这样可以使得 gtags 的使用非常便捷。

我把 f4 绑定为打开 eshell。Emacs 本身支持 shell 和 eshell，其中前者的使用并不是
很方便，比如你想要执行上一条命令你需要使用 M-p，而不能使用上箭头（同样下一条命令
需要用 M-n），而后者方便很多。eshell 是用 Emacs lisp 写的，所以和 Emacs 的融合比
较好，不存在上面的问题。此外因为它是 Emacs lisp 语言编写的，你可以在 eshell 中执
行 Emacs 命令，不过应该很少会有人这么干，因为 smex 已经足够强大了。


* * *

完
