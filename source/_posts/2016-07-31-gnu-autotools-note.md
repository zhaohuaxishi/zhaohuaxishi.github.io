---
title: GNU 的自动构建工具 autotools 小结
date: 2016-07-31 14:44:58
tags:
- 读书笔记
---

自动构建工具由来已久，使用的最为广泛的有两种，GNU 的 autotools 和 llvm 的 cmake
。这篇文章主要对于 autotools 做一个简单的小结，有机会升入学习 cmake 的 话，再补
充。

# 书籍推荐

这方面的书籍，我只看过一本，Alexandre Duret-Lutz 的《Using GNU Autotools》。严
格来说这不能算是一本书，它是一个PPT，内容非常浅显易懂，推荐给有兴趣深入了解该技
术的人。

# 最主要对两个工具

autotools 并不是一个软件的名称，它是多个软件对集合，其中最重要对两个软件是
autoconf 和 automake。

# 我们可以得到什么

通过自动构建工具，我们最终会得到两种东西：config.h 和 Makefile 前者是编译环境相
关的宏定义，而后者是完成软件编译的编译脚本。

# 如何得到这些这两种文件

这两种文件都是通过固定的模板加上具体的配置得到，config.h 的模板是 config.h.in
而 Makefile 的模板文件是 Makefile.in。而从模板到具体文件输出要做的就是我们熟悉
的 ./configure。

```
config.h.in + configure ==> config.h
Makefile.in + configure ==> Makefile
```
那么这两个模板和 config.h.in 和 Makefile.in 又是如何得来的呢？自动构建的神奇之
处就在于自动生成，而这两个模板文件正是自动生成而来。完成这一操作的命令是
autoreconfig。

# autoreconfig

autoreconfig 是一条非常神奇的命令，它是整个 autotools 中最为重要的命令之一 。大
多数的情况下，你只需要简单的 autoreconfig --install 就可以完成大部分的工 作。

当然 autoreconfig 再强大也不可能强大的到无中生有，它依旧需要一些输入，这也就是
需要我们自己编写的两个文件：configure.ac 和 Makefile.am。实际上，configure.ac
这个文件会产生两个输出，configure 和 config.h.in 而 Makefile.am 则会产生
Makefile.am.in

```
configure.ac + autoreconfig ==> config.h.in, configure
Makefile.am  + autoreconfig ==> Makefile.in
```
# configure.ac 与 宏

我们写在 configure.ac 里面的东西，大部分都是大写字母，之所以这样写是因为其实其
中的大部分内容都是宏，而宏的命名规范通常都是使用大写字母。

实际上Autoconfig 的核心是 autom4te, 它是 M4 宏处理器的驱动。所以实际上我们 写在
configure.ac 中的宏最终的处理者是 M4，我们可以在 configure.ac 中使用 为M4编写的
宏代码。比如你可以在Autoconfig Archive 下载 `ax_cxx_compile_stdcxx_11.m4`和
`ax_cxx_compile_stdcxx.m4`代码，放到项目根目 录下的m4子目录下面，然后在
configure.ac中使用AC_CONFIG_MACRO_DIR([m4])引入该文件。这样就可以在你的
`configure.ac` 中使用

```
AX_CXX_COMPILE_STDCXX_11()
```

来检测你的编译器是否支持 C++11，并在支持 C++11 的时候给你的编译器加上类似
-std=c++11 这样的编译选项


