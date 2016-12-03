---
author: 郭荣飞
date: 2015-03-28
title: 关于 C++ 参数使用指针还是引用的总结
tags:
 - C/CPP
---


# 问题#

在 C 语言中，所有的参数传递都是值传递，所以如果你需要在一个函数中改变函数外变量
值，你需要把函数的参数声明为指针（全局变量另当别论）。但是在 C++ 中存在传递引用，
它也可以用来改变变量值。此外引用也同时消除了拷贝对象带来的开销。

既然传递指针和引用都能到达到同样的效果，那么函数声明的时候应该使用引用呢还是
指针呢？

<!--more-->

# More Effective C++ 一书的总结

这本书的第一条就是区分指针和引用。它们两者之间的最大区别是引用必须指向某个对象而
指针可以是`NULL`，此外引用一旦指定不能更改而指针可以。

这两个区别点导致引用有更加安全和高效的特性，但是指针却有无可比拟的灵活性。大部分
人出于安全性的考虑会推荐使用引用，这其实也是它设计的主要目的，但是如果你想要灵活
的设计，大部分时候你只能选用指针，比如设计模式种的大部分设计都是使用指针而不是使
用引用。引用在参数传递的时候用得多一些，而类内部的组合中可能会使用指针来提高设计
的灵活性（毕竟一旦设定就无法改变对于灵活性来说是个灾难）。

# 异常安全

指针还存在的另一个优势是可以使用它实现 `pimpl`，这种手法可以达到很好的异常安全
性。引用在交换的时候实际上交换的是引用的内容，所以无法做到这一点。详见《More
Exceptional C++》一书的第22条。

# 网上的讨论#

这个问题在 `stackoverflow` 中有非常多的讨论。下面是一些链接：

   - [Are there benefits of passing by pointer over passing by reference in C++?][1]

   - [Pass by Reference v. Pass by Pointer — Merits?][2]

   - [When to pass by reference and when to pass by pointer in C++?][3]

   - [Pointers vs References: A Question on Style][4]


   [1]: http://stackoverflow.com/questions/334856/are-there-benefits-of-passing-by-pointer-over-passing-by-reference-in-c
   [2]: http://stackoverflow.com/questions/3621792/pass-by-reference-v-pass-by-pointer-merits
   [3]: http://stackoverflow.com/questions/3613065/when-to-pass-by-reference-and-when-to-pass-by-pointer-in-c/3613109#3613109
   [4]: http://www.thecodingforums.com/threads/pointers-vs-references-a-question-on-style.284603/

# 网上观点的总结#

0. 尽量避免使用指针，可以用引用的时候尽量不要使用指针。

1. 指针参数可以在调用的时候传递 `NULL` 而引用则不可以。所以如果你的参数是可选的话
   选择传递指针。

2. 指针参数在调用的时候会比引用要明显一些：

		int fun(val);
		int fuc(&val);

   前者比较难以看出是传递引用还是直接传递值，而第二个很明显是传递指针。

3. 如果你需要在函数中重新绑定改变参数，你只能用指针。因为你没有办法重新绑定一个
   引用。不过需要这么做的情况好像比较少。

4. 如果你的参数需要传递数组的话，你只能使用指针。

5. 其他情况下尽可能的使用引用。因为引用从语义上来说更直白一些，也更不容易出错。
   引用一定是指向一个合法的对象，而指针需要在使用之前检查是否为 `NULL`

6. 还有一些人觉得参数的传递如果是传递引用的话只使用 `const refercence`，把引用的
   作用限制在避免参数拷贝的开销上。然后把改变变量内容的任务交给指针。这也是一个
   非常不错的建议。
