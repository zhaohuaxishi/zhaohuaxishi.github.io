---
author: 郭荣飞
date: 2015-10-06
title: 利器系列之 —— 编辑利器 Vim 之插件配置
tags:
 - Vim
---

在每个程序员的心里都有一款完美的 IDE，只不过不同的程序员心中对于完美的定义
并不相同，所以从来都没有一款大家都喜欢的 IDE 存在，它们总是少一些你想要的
功能，或者是多了一些你不想要的功能。

解决这种问题的方法之一是配置，这也是为什么备受大家推崇的各种编辑器或者 IDE
都含有大量的可配置选项，通常来说我们可以通过配置选项把编辑器现有的功能配置
成我们最顺手的状态。

但是如果编辑器没有你想要的功能，配置是无济于事的。增强编辑器的功能，靠的是
这篇文章中要介绍的 —— 插件。插件的存在是 Vim 和 Emacs 这类的编辑器能够备受
推崇的原因之一，因为它们让不可能变成可能。

其实最理想的状态应该是像 `Shougo` 这类的大神一样，不爽了自己写一个。可惜的
是大部分人没有这个能力，我们通常是用大神写好的插件。这篇文章介绍的是如何使
用 Vim 的插件，而不是如何编写自己的插件。

<!--more-->

# 如何找到自己想要的插件

Vim 的插件很多，所以如何找到一个合适的 Vim 插件也是一件比较头疼的事情。最
简单的做法是使用别人的 Vim 配置文件，比如我最初使用的配置文件就是[k-vim][]
。这一类的配置文件通常是作者多年以来使用 Vim 过程中积累的经验，他们选用的
插件也通常是一些最好用的插件。

 [k-vim]: https://github.com/wklken/k-vim

另外一种方式就是去介绍 Vim 插件的网站上查找排名最前的那些插件。我最喜欢的
Vim 插件网站是 [vimawesome][]。这个网站上的每个插件都会有评分，一般来说最
靠前的都是大家用的最顺手的。

 [vimawesome]: http://vimawesome.com/

当然你也可以去 `github` 上搜索相关的插件，然后看看哪一个插件的 `star` 最
多。

# 插件的安装

通常我们会使用大量的插件，这些插件的管理（安装、删除、升级）非常不便。为了
解决这个难题，我们需要一个管理插件的插件，这一类型的插件很多，用的比较多的
有：

 -[Vundle][]

 -[Neobundle](https://github.com/Shougo/neobundle.vim)

 -[Pathogen](https://github.com/tpope/vim-pathogen)

 -[VimPlug](https://github.com/junegunn/vim-plug)

我一直都是使用 [Vundle][]，它可以非常方便的管理插件。中间两个我没有使用过在这
里不做介绍，最后一个是比较新的插件管理器也非常好用，有兴趣的可以去它们的主页上
查看相关信息。

 [Vundle]: https://github.com/VundleVim/Vundle.vim

Vundle 的安装比较简单，只需要两个简单的步骤即可：

## 1. 克隆插件代码：

    git clone https://github.com/gmarik/Vundle.vim.git ~/.vim/bundle/Vundle.vim

## 2. 在配置文件中插入：

    set nocompatible
    filetype off

    set rtp+=~/.vim/bundle/Vundle.vim
    call vundle#begin()

    Plugin 'gmarik/Vundle.vim'

    call vundle#end()
    filetype plugin indent on

`Vundle` 是唯一一个我们需要手动安装的插件，安装完这个插件之后你可以通过它
帮你自动完成其余插件的安装。比如我想要安装 `delimitMate` 这插件，你只需要
做如下的配置：

    call vundle#begin()

    Plugin 'VundleVim/Vundle.vim'
    Plugin 'Raimondi/delimitMate'

    call vundle#end()

也就是在 `call vundle#begin()` 和 `call vundle#end()` 之间加入你想要配置插
件的名字。Vundle 支持多种插件源，其中 `Raimondi/delimitMate` 这中写法表示
安装 `github` 上 `Raimondi` 用户的 `delimitMate` 插件。详细的语法请参考
`Vundle` 在 `github` 主页中的介绍，或者通过 `h vundle` 查看帮助文档。

# 常用插件推荐

Vim 好用的插件太多，我一般只用 Vim 做 C/C++ 的开发，所以这里推荐的插件很多
都是和 C/C++ 相关，如果你用它做其他类型的开发，可以自行寻找相关的插件。

插件安装之后通常需要一些简单的配置，这里并没有给出这些配置，因为配置通常因
人而异，不同的人喜欢不同的快捷键，写在这里也会使得文章本身变得复杂。大部分
的插件的配置方式在项目主页上都会给出，我给出了所有插件的主页链接，所以就不
再给出配置方法。如果你确实需要参考，可以参考我的[代码仓库][repo]中的相应配
置文件。

[repo]: https://github.com/zhaohuaxishi/dotfiles

## 界面美化类型插件

### [vim-colors-solarized][vimsolar]

`solarized` 是一个非常有名的主题，这个插件是它的 `Vim` 版本。 不过这个主题
只能用在 GVim 中。如果你在终端中使用 Vim，那么你应该寻找终端主题工具而不是
`Vim` 主题插件。`xfce` 自带的 `xfce4-terminal` 和 `kde` 自带的 `konsole`
都带有这个主题，`Ubuntu 15.04` 以后的 `gnome-terminal` 也带有这个主题，在
之前的 `Ubuntu` 版本中想要使用合格主题可以安装
[gnome-terminal-colors-solarized][gtsolar] 这个插件。通常这个插件需要配合
[dircolors-solarized][dirsolar] 这个插件同时使用，这样才能达到最佳的显示效果。

[vimsolar]: https://github.com/altercation/vim-colors-solarized
[gtsolar]: https://github.com/Anthony25/gnome-terminal-colors-solarized
[dirsolar]: https://github.com/seebi/dircolors-solarized

### [vim-airline][airline]

这个插件是一个状态栏（statusline）和标签栏（tabline）的一个增强插件，它是
`vim-powerline` 的一个后继插件，它使用纯 VimL 编写，不需要用到 `Python` 所
以它的速度要快一些，也轻巧一些，这也是称之为 `air` 的原因。

这个插件可以说是漂亮的不像实力派，它功能非常强大，同时又有大量的主题存在，
可配置性非常的高，强烈推荐。

[airline]: https://github.com/bling/vim-airline

### [tmuxline][tmuxline]

如果你喜欢把 `Vim` 和 `tmux` 配合起来使用，那么你可以考虑安装 `tmuxline` 这个
插件，它可以把 `tmux` 和 `Vim` 的状态栏设置成统一的主题，让这两者更加完美的融
合。关于 `tmux` 的使用，你可以参考我的另一篇博文[分屏利器 Tmux][tmuxblog]

[tmuxblog]: /2015/08/15/tmux-tutorial/
[tmuxline]: https://github.com/edkolev/tmuxline.vim

## 自动补全插件

自动补全插件通常分为两种：代码块的插入和输入自动补全。很多人想到的自动补全
都是第二种，但其实第一种的功能也是强大到没有朋友的。

### 代码块的插入

所谓代码块的插入，其实就是把一些常用的代码块通过简单的几个字符扩展成完整的
代码，相当于是缩写替换成全名。比如输入

    main

使用扩展，立刻插入：

    int main(int argc, char *argv[])
    {
        |<- 光标位置
        return 0;
    }

输入：

    fori

使用扩展，可以插入：

                 光标位置
                    V
    for (int i, i < |; i++) {
    }

这一类的代码，在你是编写任何一种语言的代码时都会存在，而这种插件的存在会大大的
提升你的编码效率。

#### [ultisnips][]

代码块的插入其实又分开为两个插件，`引擎` 和 `代码块描述`。`引擎` 的作用是
驱动整个代码块扩展，比如你输入 `main` 之后按下扩展键，它负责找到相应的代码
块并完成扩展。而 `代码块描述` 用来说明 `main` 到底是扩展成

    int main(int argc, char *argv[])

还是：

    int main(void)

[ultisnips][] 就是所谓的 `引擎`，它是基于 Python 的，所以我们需要首先安装
`Python`，在大部分的 `Linux` 发行版本中，默认自带 `Python`，其他操作系统可能需
要自行安装。

[ultisnips]: https://github.com/SirVer/ultisnips

#### [vim-snippets][]

通常我们不会自己编写 `引擎` 但是我们可以自己编写想要的 `代码块描述`。当然有一
些 `代码块描述` 写的比较好而且非常全面，所以也成了一个常用的插件，这也就免去了
自己动手写 `代码块描述` 的麻烦，[vim-snippets][] 就是这种代码块描述插件。

当然这并不意味着你就只能使用它们的 `代码块描述`，你完全可以根据自己的喜好
进行自己的定制和扩展。

[vim-snippets]: https://github.com/honza/vim-snippets

## 输入补全插件 —— [YouCompleteMe][]

这一类的插件，几乎所有的 IDE 都有提供，比如你输入 `vim-` 的时候列出
vim-`colors-solarized`，vim-`airline`，和 vim-`snippets` 供你选择补全。

这种插件在 Vim 中比较多，最受欢迎的应该是 `YouCompleteMe`，这个插件也是强
大到没有朋友的级别，不过这个插件比较难以安装，因为这个插件非常的庞大，在墙
内安装它非常的耗时，很多时候你可能会安装到一半的时候突然出错。不过这个插件
着实强大，不妨多试几次。

除了 `YouCompleteMe` 之外还有其他的一些自动补全的插件也比较好用，比如
`neocomplete` 插件。这些插件通常可以和前面的 `ultisnips` 配合使用，具体的
配置方法你可以参考它们的帮助文档，或者直接 `Google`。

[YouCompleteMe]: https://github.com/Valloric/YouCompleteMe

## 兼容插件 —— [supertab][]

`ultisnips` 和 `neocomplete` 都使用 `tab` 作为触发按键，所以会有冲突，你可
以通过配置改变其中一个的触发按键，也可以通过 `supertab` 这个插件，让它们共
存。`YouCompleteMe` 的文档说它集成了 `supertab` 的功能，如果你选择使用它的
话，可以不用安装这个插件。

[supertab]: https://github.com/ervandew/supertab

## 括号匹配

在 C/C++ 中使用了大量的括号（这里的括号包括单双引号），这些括号通常是必须
配对的，初学者容易因为错漏这些匹配的括号而导致错误。解决这一类的错误的最佳
方式通常是成对的输入这些括号，你可以手动完成他们，也可以通过插件自动完成它
们。


### [delimitMate][]，[auto-pairs][]

这个插件可以很轻松的完成括号的匹配输入，通常有它就够用了。它比较轻巧，速度
很快。另外一个类似的插件 `auto-pairs` 也可以完成这一操作，不过它的速度要比
`delimitMate` 慢一些，不过功能相对强大一些，比如：

    for (int i = 0; i < 10; i++) {|}

使用 `delimitMate` 完成匹配后输入回车，你可以得到

    for (int i = 0; i < 10; i++) {
    }

而使用 `autopair` 你可以得到

    for (int i = 0; i < 10; i++) {
        |
    }

总的来说，后者相对智能一些，不过速度不如前者，选择哪个全看个人洗好。

[delimitMate]: https://github.com/Raimondi/delimitMate
[auto-pairs]: https://github.com/jiangmiao/auto-pairs

### [rainbow_parentheses][rainbow_parentheses]

括号匹配在数量较少的时候比较容易，一旦嵌套过深，查找对应的另一半会比较困难
。 Vim 为了解决这个问题提供了 `%` 这个快捷键，跳转到对应的括号上面，这个按
键的问题在于你需要先把光标移动到其中一半括号上，而且这个按键本身不是很好输
入。`rainbow_parentheses` 这个插件的作用是把匹配的括号通过同一种颜色标出来
，这样我们可以很容易的通过视觉找到匹配的括号。这在类似于下面这样的代码中还
是比较实用的。

    if ((c = getchar()))

当然如果你的代码嵌套过深，你首先应该想的是是否该改一改你自己的代码了。

[rainbow_parentheses]: https://github.com/kien/rainbow_parentheses.vim

### [vim-surround][]

老实说这个插件和括号的匹配本身没有太大的关系，只是功能类似所以放到一块。这
个插件其实也是一个杀手级的插件，它可以在你指定的内容周围环绕成对的符号。比
如，你可以快速把

    hello world

变成：

    "hello world"

或者更神奇的：

    <H1>hello world</H1>

而这一切并不需要你切换到输入模式才能完成。关于它的详细用法可以参考它的帮助
文档。

和 `vim-surround` 配对的另一个插件是 [vim-repeat][] 有了这个插件你可以重复上
一次 `surround` 动作。

[vim-surround]: https://github.com/tpope/vim-surround
[vim-repeat]: https://github.com/tpope/vim-repeat

## Git 集成 —— [fugitive][]

Git 是我用的最多的版本控制工具，Git 和 Vim 这两个看似风马牛不相及的神器同
样可以通过插件统一起来，这个插件就是 `[fugitive][]` 有了它你可以在 Vim 中
可视化的完成大部分 Git 操作，非常的方便。

不过在 github 的项目主页上没有太多关于它的用法的介绍，建议感兴趣的人查看它
的 help 文档，里面有详细的功能介绍。

[fugitive]: https://github.com/tpope/vim-fugitive

## 多行编辑 —— [multiple-cursors][]

这个功能不知道是不是从 `sublime` 中学过来的，我在 `sublime` 中也见过，算是
它的王牌功能之一。要在 Vim 中使用这个功能你需要安装 `vim-multiple-cursors`
这个插件。这个插件最实用的地方就是你如果想要替换一个变量名，你不需要挨个的
替换，使用它可以先可视化的选择然后一次全部替换。

[multiple-cursors]: https://github.com/terryma/vim-multiple-cursors

## 光标的快速移动 —— [easymotion][]

`hjkl` 已经让光标的移动非常的方便了，但是它们毕竟只能移动一个单元格，如果
你想要移动多个单元格你需要在这些快捷键前面加上移动的数量，比如 `10j` 向下
移动 10 行，可是我很少使用这个操作因为我没有这个脑力去计算目标行和当前行之
间的距离。

为了解决这个问题，很多人推荐使用相对行号，这样就可以直观的感受到这个距离而
不用自己计算。这当然是一个不错的选择，只不过有些人不喜欢用行号，因为它占用
了一部分可用的编辑区域。此外相对行号在调试程序的时候不太方便，毕竟你的编译
器通常告诉你在哪一行出错，而不是相对当前行在哪一行出了错（当然这些都是个人
喜好问题，我通常直接通过在命令模式下输入行号跳转到我想要的行号）。

如果我可以先按下 `j` 然后选择要跳到哪一行而不是先想好要跳到哪一行再按下
`j` 该有多好。`vim-easymotion` 就是完成这个功能的，你可以先按下方向键然后
可视化的选择你的目标，一切就是这么 `easy`。

[easymotion]: https://github.com/easymotion/vim-easymotion

## 项目管理

如果你需要开发一个比较庞大的项目，你很容易迷失在大量的文件当中。这一小结主
要介绍大型项目管理的一些插件

### [nerdtree][]

这个插件，久负盛名，它给 Vim 提供了方便的目录功能，很多人对它爱不释手，因
为大部分 IDE 都有这个功能，它可以给你一种很熟悉的感觉。当然它会比普通的目
录浏览器要强大，因为它提供大量的快捷键，操作起来非常方便。此外它可以很方便
的完成打开和关闭，如果你有一块比较大的显示器，你可以一直开着它，如果你只用
笔记本，你也可以关闭它，需要的时候再打开。

[nerdtree]: https://github.com/scrooloose/nerdtree

### [ctrlp][]

这是另一个杀手级的插件，目录通常只是方便浏览而已，其实大部分的时候我们真正
想要的是快速找到并编辑某个文件。这一类的插件比较多，我最喜欢的是 `ctrlp`，
这个插件提供了文件查找，buffer 管理，最近使用文件查找等功能，通过扩展还可
以查找函数，非常的方便。我使用过许多类似的插件，但是这个是这个是我最喜欢的
，因为它的速度要比其他的插件快很多。

如果你喜欢捣鼓其他的插件，你可以考虑 [ctrlspace][] 和 [unite][] 这两个工具
，它们的功能也比较强大，萝卜白菜，各有所爱。其中 `unite` 的功能最为强大，
它有很多 [ctrlp][] 没有的功能，没有用过的可以考虑尝试一下。

[ctrlp]: https://github.com/kien/ctrlp.vim
[ctrlspace]: https://github.com/szw/vim-ctrlspace
[unite]: https://github.com/Shougo/unite.vim

### [ctrlsf][]

项目的开发，如果工程比较大，你可能会需要做一些简单的重构，比如更改一个变量
的名字。 如果这个变量只在一个文件中出现，你可以使用后面推荐的
[multiple-cursors][] 这个插件，一次选中所有的变量，然后全部更新。 但是如果
这个变量出现在不同的文件中（比如头文件和实现文件中都有出现），上面这个工具
就不够用了。这时候你可以使用 [ctrlsf][] 这个插件。

这个插件和 [multiple-cursors][] 一样是从 `sublime` 中学过来的功能，你可以
查找工程中某个名称出现的所有位置。如果只是单纯的查找，[ack][] 这个插件也可
以完成这一工作，[ctrlsf][] 真正强大的地方在于你可以在结果页面中进行编辑，
这个功能配合 [multiple-cursors][] 可以实现非常强大的重构功能，强烈推荐大家
使用。

[ctrlsf]: https://github.com/dyng/ctrlsf.vim
[ack]: https://github.com/mileszs/ack.vim

### [tagbar][]

许多的 IDE 会有一块区域列出当前文件中所有函数，Vim 中也可以通过插件实现这
一功能。我比较喜欢的插件是 `tagbar`，轻巧但很强大。这个插件依赖 ctags，所
以你需要在安装插件之前安装这个软件包。

[tagbar]: https://github.com/majutsushi/tagbar

### [cscope][]

如果你的项目非常的大，代码之间的跳转会变成一件非常的困难的事情，你通常在阅
读到一个函数的调用的时候想要阅读一个函数定义，或者函数的实现。在项目比较小
的时候你可以自己查找然后打开定义文件，当项目非常庞大的时候这一切就变得力不
从心了。你可能会很希望有一个插件，让你直接使用快捷键就可以跳转到该函数的实
现中去。`YouCompleteMe` 插件可以配置这个功能，你也可以使用更加强大的
`cscope`。

cscope 不是 Vim 的插件，而是 tag 的生成工具，你可以通过它和 Vim 的集成，让
它为 Vim 生成 tag，方便 Vim 进行跳转，具体配置参考 `cscope` 主页给出的方法
。

[cscope]: http://cscope.sourceforge.net/

## 代码注释

### 快速注释代码 —— [nerdcommenter][]

代码的快速注释是一个非常实用的功能，这个功能我通常使用 `nerdcommenter` 来
实现。它可以很方便的实现代码的注释和反注释的功能。

[nerdcommenter]: https://github.com/scrooloose/nerdcommenter

### 文档化代码注释 —— [DoxygenToolkit][]

好的代码注释本身就是一份非常好的文档，要把注释转换成书面文档，通常会使用
`doxygen`，我们可以通过 [DoxygenToolkit][] 这个插件很方便的在 Vim 中编写符合
`doxygen` 格式的注释文档，然后通过 `doxygen` 生成相应的文档。

[DoxygenToolkit]: https://github.com/vim-scripts/DoxygenToolkit.vim

## C++ 相关的其他插件

### 语法高亮升级 —— [vim-cpp-enhanced-highlight][cppehl]

Vim 内置的语法高亮对于 C++ 的支持并不完美。对于标准库它无法高亮显示，
[vim-cpp-enhanced-highlight][cppehl] 这个插件可以增强高亮效果。

[cppehl]: https://github.com/octol/vim-cpp-enhanced-highlight

### 头文件和实现文件之间的跳转 —— [vim-fswitch][]

如果我们处于开发的初期，接口的定义并不是特别完善的时候，我们通常是需要在头
文件和实现文件之间不断的跳转的。你很可能在实现一个接口的时候突然发现需要修
改它的接口定义，于是需要找到头文件修改定义，又或者你在头文件中修改了接口的
定义，需要去实现文件中做相应的更新。[vim-fswitch][] 这个插件可以帮我们完成
这一工作。

[vim-fswitch]: https://github.com/derekwyatt/vim-fswitch

### 代码格式化工具 —— [vim-autoformat][]

Vim 的代码格式化功能个人觉得只能算是凑合，C++ 代码格式化的选项并不多，为了
能够更好的格式化 C++ 代码，个人比较倾向于使用 [astyle][]。

如果你需要在 Vim 中使用 `astyle` 你可以考虑安装 [vim-autoformat][] 这个插
件，它可以在 Vim 中集成各种格式化工具包括 `astyle`。`vim-autoformat` 插件
需要 `astyle 2.0.5` 以上的版本，`Ubuntu` 中的版本没有达到这个要求，所以你
需要自己编译安装它。

[vim-autoformat]: https://github.com/Chiel92/vim-autoformat
[astyle]: http://astyle.sourceforge.net/

****

Vim 的插件太多，以上只是个人常用的一些而已，你可以自己去寻找自己喜欢的那些
插件。此外插件好用不要贪杯，多了会让你的 `Vim` 失去它原本的轻巧和敏捷。
