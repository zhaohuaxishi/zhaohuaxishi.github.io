---
title: C++11 智能指针的常见问题
date: 2016-12-05 13:13:30
tags:
 - C/CPP
---

C++ 之父说 C++11 像是一门新的语言，很大一部分得益于标准库在 C++11 中的重要扩充，
而新的智能指针属于其中至关重要的一点。新的智能指针`shared_ptr`和`unique_ptr`比起
`auto_ptr`要好用很多，但是依旧存在一些比较容易踩坑的地方，本文罗列了
stackoverflow 上关于智能指针的一些常见问题。

<!--more-->

# 使用 shared_ptr 还是 unique_ptr

# 多继承和智能指针

[Using shared_ptr with multi inheritance class][multi-inherit]

[multi-inherit]: http://stackoverflow.com/questions/23872910/using-shared-ptr-with-multi-inheritance-class

# 指向 this 的智能指针

[std::shared_ptr of this][this]
[this]: http://stackoverflow.com/questions/11711034/stdshared-ptr-of-this

# 为什么库中比较少用智能指针

[Why do C++ libraries and frameworks never use smart pointers?][not-in-lib]
[not-in-lib]: http://stackoverflow.com/questions/10334511/why-do-c-libraries-and-frameworks-never-use-smart-pointers

# unique_ptr 无法使用类的前缀声明
