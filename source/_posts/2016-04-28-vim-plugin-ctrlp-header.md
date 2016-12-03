title: 我的第一个Vim插件 —— ctrlp-header
date: 2016-04-28 09:22:27
tags:
 - Vim
---

这是我自己编写的第一个`Vim`插件，写这个插件的原因是在编写 C++ 程序的时候，引入
头文件需要离开当前位置，跳到文件头部，找到合适位置插入 `#include <xxxx>` 这样的
语句，然后返回到原本编辑的位置继续编辑。这是一个重复性非常强的操作，所以我想尝
试编写一个插件来完成这个操作。`jetbrains`系列的`IDE`中有一个 `Alt-Enter`自动引
入头文件的功能，我最终想要实现的效果就是这个样子。

本人平时是 [CtrlP][ctrlp] 这个插件的重度用户，也知道它可以进行扩展，所以自己参照
[ctrlp-funky][funky] 中的代码写了这个 [ctrlp-header][] 插件。

[ctrlp]: https://github.com/ctrlpvim/ctrlp.vim
[funky]: https://github.com/tacahiroy/ctrlp-funky
[ctrlp-header]: https://github.com/zhaohuaxishi/ctrlp-header


目前的功能非常简单，效果图如下：

# C++ 文件

![ctrlp-header-demo][democpp]

[democpp]: /img/posts/ctrlp-header-cpp.gif

# C 文件


![ctrlp-header-demo][democ]

[democ]: /img/posts/ctrlp-header-c.gif

# 基本原理

这个插件非常简单，目前的功能非常粗糙，只是从 `en.cppreference` 中获取标准库中的
头文件列表，提供给 ctrlp，当用户选择了某一个头文件的时候，在当前的 `buffer` 中
找到合适的位置插入 `#include <header_user_selected>` 就可以了。

# 后续工作

这个插件还有很多工作没有完成：

[ ] 支持用户编写的头文件

[ ] 通过记录用户曾经的操作来提供最佳的候选项目

[ ] 支持语义解析，准确的提供头文件
