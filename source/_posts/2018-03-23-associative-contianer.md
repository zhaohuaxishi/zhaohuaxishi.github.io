---
title: C++轮子——STL关联容器
date: 2018-03-23 21:05:36
tags:
 - C/CPP
 - C++轮子
---

上一篇文章中我们简单的介绍了一下STL中的序列容器和容器适配器，这篇文章中我们将重点介绍STL中的关联容器（最后两个在概念上应该不是关联容器，但是因为和前面的容器联系太紧密，统一放在这里讲解），主要内容包括：

- std::set
- std::map
- std::multi_map
- std::multi_set
- std::unordered_map
- std::unordered_set

<!--more-->

# std::set

通常提到管理容器，大家可能首先会想到的是`std::map`，这里先讲解`std::set`是避免出现`关联数组`和`关联容器`概念上的混淆。前者是数据结构中的概念，而后则是C++中特有的概念。

## 什么关联容器

关联容器这个概念，在C++里面是一个concept（关于concept的讨论我们在上一篇文章中有讲解，这里不再赘述），它的基本定义如下：

> An AssociativeContainer is an ordered Container that provides fast lookup of objects based on keys.

翻译成中文，大概是说管理容器是提供了基于key的快速访问的有序容器。它实际上是 Container 这个 concept 的 refinement。

## Refinement

Refinement 在泛型编程中是一个非常重要的概念，它和 concept 的关系类似于OO世界中父子集成关系。我们知道 Concept 表示的不是一个类型而是一个类型集合，所以 Refinement 表示的其实也是一个类型集合，这些类型比 Concept 中的定义的类型要有更多的约束和特性（约束和特性其实是一个问题的两个方面，你约束的越是严格，你能够使用的特性越多，但是你能使用的场景越少，比如 Java 中 Object 类哪儿都能用，又哪儿都用不了）。

## 有序

在数学概念上，集合表示具有某种特定性质的事物的总体，它有三个特性：无序性、互异性、确定性。

所以个人认为，C++世界里面的`std::set`其实并不是传统意义上的集合，因为它不符合无序的特性（当然你也可以理解成无序表示任何顺序都可以，有序也属于无序的一种状态）。

## 互异

互异是指集合内部的任意两个元素都不相同，这一点在`std::set`中满足，但是在`std::multi_set`中并不满足。

`std::set`元素的互异在编程中有一个非常大的作用是去重，当然你也可以使用sort+unique的方式来处理去重，但是`std::set`的方式在逻辑上更具有表达力。

# std::map

std::map可能是C++世界使用得最广泛的关联容器，它以键值对的形式存储数据。

# std::multi_map

# std::multi_set

# std::unordered_map

# std::unordered_set
