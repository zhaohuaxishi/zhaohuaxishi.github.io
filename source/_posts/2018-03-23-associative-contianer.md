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

## 成员方法

std::set中的很多成员和std::map类似，只不过它们的value_type不一样，`std::set<std::string>`的value_type是std::string，而`std::map<int, std::string>`的value_type是`std::pair<int, std::string>`。因此这一小节不介绍相关的成员函数，你可以查看std::map的相关成员函数的介绍，然后对照std::set的[文档](http://devdocs.io/cpp/container/set)来理解这一部分内容。

# std::map

std::map可能是C++世界使用得最广泛的关联容器，它以键值对的形式存储数据的一种关联容器。

## 关联数组

在数据结构的范畴里面，map又被称为关联数组。一个普通的数组，可以使用int（严格来说应该是size_t）做为下标来访问数组内部的元素，关联数组则可以使用任意类型做为下标来存取数组内部的数组。当然这里说的数组是一个逻辑上的概念，关联数组的实现物理上的实现可能是树或者哈希表等等。

需要注意的是关联数组和关联容器是完全不同的两个概念，关联容器强调的是提供快速访问的有序容器，这个概念只存在于C++中；而关联数组强调的是键值对的形式存储数据，并通过key来快速访问数据，这个概念是传统数据结构中的概念，在不同的语言对应不同的类型，比如Java的map，Python的dict。关联容器不一定是关联数组，比如set；关联数组也不一定是关联容器，比如unordered_map。

## 红黑树

map和set的实现通常是一个颗红黑树，红黑树是一种平衡二叉排序树。它的节点要么是黑要么是红（我也很好奇为什么不是要么黑要么白），但是如果节点是红，它的两个子节点一定是黑，而且任意节点到叶子节点的任意路径中的黑色节点一样多。这个特性可以保证树的两个分支深度最多只会出现2:1的关系（全黑和红黑相间），所以它基本平衡，它不完美，没有AVL好看，但是它的实现比较简单，出现不平衡的时候，调整比较简单，所以在实际运用中使用的频率很高（比如Linux内核的进程调度用的就是红黑树）。

# 数据的添加

往map里面添加数据有三种方式：operator[]，insert，emplace。效率上来说，通常后面的效率会高于前面的效率。emplace 和 insert 的区别我们在[C++轮子 —— STL 序列容器](/2018/02/25/stl-part-two-vector/#push-back-vs-emplace-back)一文中有提到过，这里不再重复解释。

理论上来说，operator[]实际上是数据访问的方式，但是因为它返回的是引用，所以可以直接用于改变元素的值。如果 key 不存在，operator[]会先调用`mapped_type`的默认构造创建一个空的对象（如果你确保数据存在，可以考虑使用std::map::at，如果key不存在它会抛异常）。比如一个统计单词出现次数的程序可以这样写：

```
std::map<std::string, std::size_t> words;

std::istringstream iss(str);
std::string word;
while (iss >> word) {
    words[word]++;
}
```

上面这段代码之所以能够成功运行，是因为如果单词第一次出现，words[word]会使用std::size_t的默认构造（std::size_t()）创建一个新的元素并返回，这样一来新的单词出现次数可以做正确的累加。

同样因为这个特性你不应该使用operator[]来判断数据是否存在，比如你不能像下面这样去判断一个元素是否存在。

```c++
if (words["word_not_exist"]) {
    //
}
```

因为如果元素不存在，它会创建一个新的对象，如果你只是想判断元素是否存在，这种行为通常都不是你想要的。

你实际上需要的是功能是`Contains`，正确的写法是使用`std::map::find`函数：

```
auto iter = words.find("word_not_exist");
if (iter != std::end(words)) {
    //
}
```

operator[]在mapped_type是一个比较重的类型的时候，也可能造成不必要的开销。因为它会县构建一个空的对象，随后又被特换掉，比如你有一个map用于映射人名和职位名称。

```
std::map<std::string, std::string> employee;
employee["bob"] = "software enginer";
```

这个地方先用"bob"构造了std::string，然后用这个string作为key创建了一个空字符串作为value，然后返回这个vlaue的引用，最后调用这个value的operator=完成赋值。实际上中间的空的字符串完全可以不需要构造，我们可以直接通过调用`insert`方法来插入数据。

```
employee.insert(std::make_pair("bob", "software enginer"));
```

需要注意的是map的value_type是std::pair<std::string, std::string>而不是std::string. 如果你不想构建这个pair，你可以使用`emplace`。

```c++
employee.emplace("bob", "software enginer");
```

## find

前面在谈到`Contains`的时候我们有提到，std::map::find，之所以单独提到它是因为序列容器的查找通常使用的都是算法：std::find，而关联容器通常都提供`find`成员，你当然也可以用std::find，但是它的复杂度是O(N)，而`find`成员的复杂度是O(logn)。通常如果一个容器提供了某个算法的同名成员函数，你都应该应该优先使用成员函数，因为它们的效率更高，获取无法使用算法实现（比如 std::map::count，std::list::erase等）。当然换句话说，如果你找不到某个你想要的成员函数的时候，你可以去算法库里面找找。

## lower_bound，upper_bound，equal_range

关联容器的核心特点是元素有序，所以可以快速的查找数据。它不仅可以查找数据是否存在，它还可以用于查找给定元素的上下界区间。你可以使用lower_bound函数查找到给定元素的下界，也就是容器中第一个不小于指定元素的位置；同样你可以使用upper_bound来找出到给定元素的上界，也就是容器中第一个大于给定元素的位置；而equan_range可以用于返回前两者构成的区间。

定义上可能比较绕，我们举一个具体的例子：

```
const std::map<uint32_t, uint32_t> kBitratesOfResolution = {
        {192 * 144, 400},       // 144p
        {320 * 240, 500},       // 240p
        {480 * 360, 600},       // 360p
        {640 * 480, 900},       // 480p
        {1280 * 720, 1800},     // 720p
        {1920 * 1080, 30000},   // 1080p
};

kBitratesOfResolution.lower_bound(1280*720);    // 720p 指定的那个位置
kBitratesOfResolution.upper_bound(1280*720);    // 1080p 所指的那个位置
kBitratesOfResolution.equal_range(1280*720);    // [720p, 1080p]
```

# std::multi_set

# std::multi_map

# std::unordered_map

# std::unordered_set
