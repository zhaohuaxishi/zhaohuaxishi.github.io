---
title: 浅谈字符字面量的地址
date: 2018-12-15 09:25:43
tags:
  - C/CPP
---

在C和C++的世界里，我们允许字面量直接出现在表达式中，常用的字面量分为两种：字符字面量和数字字面量，比如：

```cpp
const char* sl = "hello world";
int a = 42;
```

在上面的例子中，`hello world`和`42`都是字面量，因为他们没有任何的变量和它对应，直接以值的形式出现在表达式中。

它们两者虽都是字面量，但是他们两者有一个非常本质的区别，那就是字符字面量有自己的地址而数字字面量没有。个人理解的原因是，C/C++语言中并没有内置字符串这种类型，字符字面量实际上是`char`这种内置类型的一个序列，我们没有办法原子操作字符串，所有关于它的操作都需要借助指针来完成。在上面这个例子中：`int a = 42`实际上把`42`这个值给了`a`这个变量，而`const char* sl = "hello world"`则是把`"hello world"`这个字面量在内存中的地址，给了`sl`这个指针变量【1】。

那么，同一个字符字面量的值，地址会是一样的吗？也就是说下面这个例子的输出到底是s什么：

```cpp
const char* s1 = "Hello world";
const char* s2 = "Hello world";

std::cout << std::boolapha << (s1 == s2) << std::endl;
```

<!--more-->

这个问题，一开始看上去很容易认为是`true`，但实际上，上面这个程序的输出是未定义行为，也就是说结果因编译器而异。下面这段描述来自`cppreference`

> The compiler is allowed, but not required, to combine storage for equal or
> overlapping string literals. That means that identical string literals may or
> may not compare equal when compared by pointer.

翻译成中文：

> 编译器可以，但是不要求，把相同或者重叠的字符字面量存放在一起。这意味着，同一个字符字面量在用指针做比较的时候，得到的结果并不一定相等。

这也是为什么上面例子是未定义行为的原因。在我们测试过的编译器中，假如`s1`和`s2`定义在不同的文件中，他们的值是不一样的。



----

【1】如果写成 `char s1[] = "hello world"`则是把字面量的内容放到数组`s1`对应的内存中
