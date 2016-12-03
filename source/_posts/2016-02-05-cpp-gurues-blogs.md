title: C++ 学习资源的汇总
date: 2016-02-05 16:34:57
tags:
 - C/CPP
---

我大学的时候学《论语》，老师给我们说了一句至今让我记忆犹新的话：如果你想要成为国
学大师，你需要有大师在你的身边指引。 C++是一门比较难以精通的语言，想要成为大师需
要有大师的引导。很不幸的是大师可遇而不可求，幸运的是网络时代让我们离大师越来越近
，甚至触手可及。接收大师的熏陶的一种方式是拜读他们的作品，你可以去读他们的书或者
你也可以去读他们的博客。

<!--more-->

# C++ 相关的博客/网站/文摘

- [Bjarne Stroustrup](http://www.stroustrup.com/)：C++语言之父
- [Scott Meyers](http://www.aristeia.com/)：《Effective》系列书籍作者
- [Stan Lippman](http://blogs.msdn.com/b/slippman/)：《C++ Primer》、《深度探索
  C++对象模型》、《Essential C++》的作者
- [Herb Sutter](http://www.gotw.ca/)：《Exceptional》系列书籍作者
- [isocpp](https://isocpp.org/)：C++ 标准

国内不错的C++文摘：

- [伯乐在线](http://blog.jobbole.com/category/c-cpp/)
- [酷壳](http://coolshell.cn/category/proglanguage/cplusplus)

# C++ 必读书籍

在 `Stack Overflow` 上有一个非常火的问题：[The Definitive C++ Book Guide and
List][enaddr] 上面罗列了大量的C++精品书籍，这里是一份中文翻译（翻译原文出自[计算
机书籍控 ][chaddr]，但是翻译比较陈旧，我参照目前的答案原文进行了更新）：

 [enaddr]: http://stackoverflow.com/questions/388242/the-definitive-c-book-guide-and-list
 [chaddr]: http://bestcbooks.com/recommended-cpp-books/

## 初级

**如果你是一个无编程经验的C++初学者**

- 《C++程序设计原理与实践》(Programming: Principles and Practice Using C++ )作
  者： Bjarne Stroustrup（**更新到C++11/C++14**） C++之父写的C++入门书籍。本书面
  向没有编程经验的初学者，但相信有编程经验的人也能从本书中学到不少东西。

**如果你有其它语言经验的C++初学者**

- 《C++ Primer》作者：Stanley Lippman, Josée Lajoie, and Barbara E. Moo （**更新
  到 C++11**） 近1千页，本书透彻的介绍了C++，以浅显和详细的方式讲到C++语言差不多
  所有内容。2012 年8月发行的第五版包含C++11的内容。（*不要和 C++ Primer Plus
  (Stephen Prata) 搞混了，该书远不及前者受欢迎。*）

- 《C++ 语言导学》(A Tour of C++) 作者： Bjarne Stroustrup，这本导学书籍是一本关
  于 `标准C++`（包括语言本身和标准库，使用C++11标准）的所有方面的快速（分成14章
  ，大概 180页）入门教程。这本书站在一个相对较高的级别向已经了解C++或者至少是有
  经验的程序员介绍C++。这本书是《C++ 程序设计语言第四版》一书中2-5章内容的一个扩
  展。

- 《Accelerated C++》作者：Andrew Koenig and Barbara Moo 这本书覆盖了和C++
  Primer 一样的内容，但厚度只有C++ Primer的四分之一。这主要是因为本书面向的不是
  编程的初学者，而是有其它语言经验的C++初学者。对于初学者，本书学习曲线稍显陡峭
  ，但对于能克服这一点的学习者而言，它确实非常紧凑的介绍了C++这门语言。（历史上
  ，这是一本划时代的书籍，因为它是第一本使用现代方法向初学者教授语言的书籍）

- 《C++编程思想》(Thinking in C++)作者：Bruce Eckel 共两卷，是一本教程式的免费入
  门书籍，可惜的是这本书被充斥着许多低级错误（比如书中认为临时变量默认都是const
  ），并且没有官方的勘误列表。在 （http://www.computersciencelab.com/Eckel.htm）
  中有一份不完整的第三方勘误列表，可惜的是这份文档目前明显不再维护。

**最佳实践**

- 《Effective C++》作者：Scott Meyers 本书以瞄准成为C++程序员必读的第二本书籍而
  写， Scott Meyers成功了。早期的版本面向从C语言转过来的程序员。第三版修改为面向
  从类似 Jave等语言转来的程序员。内容覆盖了50多个很容易记住的条款，每个条款深入
  浅出（并且有趣）讲到了你可能没有考虑过的C++规则。关于C++11和C++14的例子和一些
  过时的信息可以参考《Effective Modern C++》。

- 《Effective Modern C++》作者：Scott Meyers 本书基本上可以看成是《Effective C++
  》一书的最新版本, 书籍的主要目的是帮助C++程序员完成从`C++03`到`C++11`和`C++14`
  的过渡。

- 《Effective STL》作者：Scott Meyers 讲解方式和Effective C++类似，但内容主要面
  向于 STL 。

* * * *

## 中级

- 《More Effective C++》作者：Scott Meyers 更多（深入）关于C++的规则。没有前一本
  Effective C++重要。但同样值得一读。

- 《Exceptional C++》作者：Herb Sutter 讲解方式为提出并解决一系列的C++难题。本书
  极其透彻的讲解了C++资源管理、异常安全和RAII。同时覆盖了一些较为深入的技术，比
  如：编译防火墙(pimpl idiom)、名字查找规则,、好的类设计和C++内存模型。

- 《More Exceptional C++》作者：Herb Sutter 讲到了Exceptional C++没有涉及到的更
  高级的异常安全技术, 同时讨论了高效的C++ OOP方式和如何正确的使用STL。

- 《Exceptional C++ Style》作者：Herb Sutter 讨论了泛型编程、最优化和资源管理。
  本书出彩之处在于谈到了如何用非成员函数和单职责原则编写模块化的C++代码。

- 《C++编程规范》(C++ Coding Standards) 作者：Herb Sutter and Andrei
  Alexandrescu “编程规范”这里并不是”代码缩进要用几个空格”。这本书包含了101个例子
  、惯用法、缺陷，通过这些可以帮助你编写正确、清晰高效的C++代码。

- 《C++ 模板完全指南》(C++ Templates: The Complete Guide)作者：David Vandevoorde
  and Nicolai M. Josuttis 本书是关于C++11之前的模板的。它覆盖了从非常基础到最高
  级的元编程知识，解释了模板工作原理的细节（概念和实现方式）。并且讨论了大量的缺
  陷。附录中包含关于ODR和重载的精彩总结。

* * * *

## 高级

- 《C++设计新思维-泛型编程与设计模式之应用》(Modern C++ Design ) 作者：Andrei
  Alexandrescu 泛型编程鼻祖级书籍。本书先介绍了基于策略（policy-based)的设计、
  type lists 和泛型编程基础， 然后讲到了许多有用的设计模式(包括small object
  allocators, functors, factories, visitors, and multimethods) 如何被高效、模块
  化、清晰的泛型代码实现。

- 《C++模板元编程》(C++ Template Metaprogramming) 作者：David Abrahams and
  Aleksey Gurtovoy 更多的是讲解boost::mpl，想要深入理解mpl的可以看一下。

- 《C++并发编程实践》(C++ Concurrency In Action) 作者：Anthony Williams 这本书主
  要内容是C++11的并发支持，包括线程库、原子(atomics)库、内存模型、锁和互斥量。同
  时也讲解了开发和调试多线程程序的一些难题。

- 《Advanced C++ Metaprogramming》 作者：Davide Di Gennaro 前C++11时代TMP技术的
  手册级书籍。本书更侧重于工程实践。里面有大量的可能几乎无人知道但很实用的技术写
  成的代码。本书可能比Alexandrescu的书更值得读。对于资深的开发者来说，这是一个学
  习C++ 暗角技术的绝佳机会，通常这些技术要通过资深的编程经历才能获取。

* * * *

## 手册类 —— 适合所有级别读者

- 《C++程序设计语言》(The C++ Programming Language) 作者：Bjarne Stroustrup(**更
  新到 C++11**) C++之父写的经典C++书籍。内容覆盖C++的所有东西，从语言内核到标准
  库、编程范式和语言哲学(这使得最新版突破1千页)。2013年5月出版的第四版涵盖了
  C++11的内容。

- 《C++标准程序库》(C++ Standard Library Tutorial and Reference) 作者：Nicolai
  Josuttis (**更新到C++11**) 这本书是C++标准库（STL）的引导和手册。 2012年4月发
  行的第二版涵盖了C++11。

- 《The C++ IO Streams and Locales》作者：Angelika Langer and Klaus Kreft 除了这
  本书，市面上基本没有讲解streams and locales的书。

C++11 手册

- 《The C++ Standard (INCITS/ISO/IEC 14882-2011)》作者：C++标准委员会 这当然是
  C++ 最权威的标准。要注意的是，C++标准是提供给有足够精力和时间的专家级用户研究
  用的。国内估计很少有人看，在国外一般它的第一个发行版也非常贵($300+ US)，国外有
  人会买现在价值$30US的电子发行版。

- 《Overview of the New C++ (C++11/14)》作者：Scott Meyers(**更新到
  C++11/C++14**) 这是 Scott Meyers开设的一个为期3天的C++课程的教材。Scott Meyers
  是C++社区最受尊敬的作者之一。虽然内容比较简短，但质量极高。

* * * *

## 经典/古老

**注意: 下列书中的部分内容可能有些过时或者不再认为是最佳实践**

- 《C++的设计与演化》(The Design and Evolution of C++ )作者：Bjarne Stroustrup
  如果你想知道为什么C++是今天这个样子，那么这本书将给你答案。本书覆盖C++标准化之
  前的一切东西。

- 《C++沉思录》(Ruminations on C++) 作者：Andrew Koenig and Barbara Moo 本书不是
  为了讲解具体的C++技术细节，而是如何通过C++编写出色的OO代码。

- 《Advanced C++ Programming Styles and Idioms》作者：James Coplien 讲解了一些
  C++ 特有的惯用法. 它确实是一本不错的书籍，如果时间闲暇也可一读。不过它确实很老
  了，可能有些不符合现代的C++。

- 《大规模C++程序设计》(Large Scale C++ Software Design) 作者：John Lakos 本书介
  绍了如何管理大规模C++软件项目的技术。很值得一读，除了有些过时以外。它是在C++98
  以前写的，缺少了好多对大规模项目重要的特性（比如名字空间）。假如你工作在一个大
  规模的 C++项目中，你可能想要读它, 不过你需要注意那些不适用甚至错误的技术点。

- 《深度探索C++对象模型》(Inside the C++ Object Model) 作者：Stanley Lippman 如
  果你想知道虚函数是如何实现、多继承时基类是如何在内存中排布的和所有影响性能的东
  西，那么这本书会给你答案。不过这本书有好多低级的拼写排版错误，英文原版错误更多
  ，侯捷翻译的版本中注明和纠正了很多，但本书绝对值得一读，你将明白编译器如何实现
  C++的对象模型。

* * * *
