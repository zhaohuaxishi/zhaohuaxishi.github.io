---
title: C++轮子——STL算法和迭代器
date: 2018-04-13 15:44:39
tags:
 - C/CPP
---

总体来说，STL包括的最重要的两大部件是容器和算法，一个泛化数据的存储，一个泛化数据的操作。前面两篇文章我们简单的介绍了STL中的容器，这篇文章介绍STL算法以及粘合容器和算法的迭代器。

# 数据和操作

计算机领域有一个非常著名的公式：

> 程序 = 数据结构 + 算法

笼统一点来说，我们可以认为：

> 程序 = 数据 + 操作

容器抽象了数据的存储，而数据的操作有一部分（比如插入删除等）直接放到来容器内部作为成员函数，另一部分则在算法库中作为一个范型算法。是否放到容器内部做为成员函数，主要是看这个操作是否和数据存储相关（还有可能和效率相关），因为算法实际上和数据是什么类型，以什么样的方式存储是无关的。

# 为什么分离算法库和容器库

下面这段定义来自《算法导论》

> An algorithm is a sequence of computational steps thats trans form the input
> into the output.

翻译成中文是说，算法是把输入变成输出的一系列计算步骤。这段定义有几个隐藏的点值得讨论：

## 1. 算法和输入数据的类型无关

当我们描述快速排序算法的时候，我们其实在描述排序的步骤，至于数据类型是`int`还是`double`和算法本身其实没有关系。这个概念在C语言中比较不好表达，通常需要通过`void*`加上函数指针来实现。比如C语言中的排序算法定义如下：

```
void qsort( void *ptr, size_t count, size_t size,
            int (*comp)(const void *, const void *) );
```

C++中有范型的存在，这个问题解决起来就方便很多，C++中的`std::sort`定义如下：

```
template< class RandomIt >
void sort( RandomIt first, RandomIt last );
```

这个定义比C的定义要简单很多也清晰很多

## 2. 算法和输入的数据是如何存储无关

我们描述一个算法，说的是数据的操作步骤，至于如何完成这些操作，实际上并没有规定。在C语言中，操作如何完成和数据如何存储有很大的关系，比如我们要实现指针的递增操作，我们的数据必须是连续存储的，而C++通过操作符的重载可以使得数据本身是如何存储的变得无关紧要。比如查找算法，在C语言中查找一个字符串中的指定字符可以使用`strchar`

```
char *strchr( const char *str, int ch );
```

而在C++中，查找算法`std::find`定义如下：

```
template< class InputIt, class T >
InputIt find( InputIt first, InputIt last, const T& value );
```

算法和数据的存储方式无关这一点，是STL中容器和算法库分离的很重的一个原因，它可以极大的提升范型算法的适用降低代码重复性。比如Java中的`ArrayList`和`LinkedList`都实现了`indexOf`方法来实现查找的功能，而C++中只实现来一个`std::find`它可以通吃`c array`，`std::array`，`std::vector`，`std::deque`，`std::list`，`std::string`等，这其实也是范型算法的魅力所在。

# 模板函数

STL是基于模板实现，容器基于模板类，而算法基于模板函数。模板函数的语法其实很简单，只要把正常的函数的参数或者返回值类型都参数化就可以了，选择两个数中的最大值，我们可以使用`std::max`：

```C++
template< class T >
const T& max( const T& a, const T& b );
```

两个参数和返回值的类型都参数化了。

## 自动类型推导

模板函数有一个非常中的特性就是它支持类型的自动推导，我们实例化一个范型类的时候需要手动指定模板参数类型：

```
std::vector<int> iv;
```

但是我们实例化一个模板函数却通常不需要，因为参数可以自动推导：

```
std::max(1, 999);
```

当然有些情况下自动推导会失败，这个时候我们可以显式指定模板函数的参数类型：

```
std::max<double>(1, 2.0);
```

## 简化模板类的创建

模板函数参数的自动推导在使用上非常方便，这一点被广泛的用于模板类的工厂方法的实现（广义上任何用于创建类的实例的方法都可以称为工厂方法）。比如我们要构造一个`std::pair`，有两种方式。

```
std::pair<int, int> point(1, 2);
std::make_pair(1, 2);
```

很显然后面这种方式用起来会比前面的方式舒服一些。标准库中存在大量的这一类型的工厂方法，比如：`std::make_tuple`，`std::make_excpetion_ptr`，`std::make_shared<T>`等等。

# 区间

前面提到算法是把输入变成输出的计算步骤，所以要写一个范型算法，首先要解决的问题是如何表达输入。一个范型算法的输入通常是由两个迭代器组成的左闭右开的区间表示的，C语言中则通常是一个首地址+长度这种方式。使用半开闭区间的方式有下面这些好处：

- 空集的概念很好表示，首尾相同即可 [beg, beg)
- 比较容易返回错误值，数据查找，如果没有找到，我们不需要返回一个特殊值，直接返回`end`，就可以来，因为`end`不在区间内部，返回它很好的表达没有找到这个概念。
- 比较容易表达迭代终止条件这个概念，`beg == end`即可表示迭代终止，这对于迭代器来说是很重要的，因为它不需要支持算数操作符，只需要支持判等符即可。

大多算法如果有输出，基本上也都是返回一个迭代器，如果成功，返回区间内的值，如果失败返回end。

## 获取区间

在C++11之前，我们通常使用容器提供的begin和end成员来获取区间：

```
std::vector<int> ia = {1, 3, 4, 2, 8};
std::find(ia.begin(), ia.end(), 3);
```

在C++11提供来范型函数`std::begin`，`std::end`，来获取区间（STL刚开始就是用这两个函数，只是加入标准库的过程中被去掉，现在又加回来了）。

```
char ia[] = {1, 3, 4, 2, 8};
std::find(std::begin(ia), std::end(ia), 3);
```

PS：数组大小的自动推导也是一个非常有意思的点，有兴趣的可以自行了解一下。

## 简化

其实每次都要调用std::begin、std::end，非常无聊，一不小心还可能输错。boost::range库可以帮我们简化这个过程：

```
std::vector<int> ia = {1, 3, 4, 2, 8};
boost::find(ia);
```

# 迭代器

算法的输入通常是一个区间，而这个区间由两个迭代器组成。迭代器是指针这个概念的泛化，和其他所有的泛型定义一样，它实际上是一个[Concept](http://devdocs.io/cpp/concept/iterator)，基本定义如下：

> The Iterator concept describes types that can be used to identify and traverse the elements of a container.

也就是说它主要提供两个功能，标识元素以及遍历容器。对于一个普通的泛型算法来说，遍历功能表现在它的参数上面，而标识功能表现在它的返回值。

迭代器作为容器（这里说的容器是广义上的容器，C数组也算是容器之一）和算法之间的桥梁而存在。但是算法的输入通常都不是迭代器这个Concept，而是这个Concept的5个Refinement。

## 迭代器的分类

迭代器虽然是指针的泛化版本，但是大部分的算法实际要求的特性会比指针少很多，为了尽可能的提升算法的适用性，迭代器的功能被拆分成5个不同的Concept：输入迭代器，输出迭代器，前向迭代器，双向迭代器，随机迭代器。

### 输入迭代器

一个指针最常见的功能就是对它取值，而这个功能组成了输入迭代器的核心概念：支持从中读取数据（这里面有一些隐含的条件，比如需要支持判等）。这是一个非常简单的功能，但是仅仅基于这个功能我们就可以写出很多有用的算法：

比如求和：

```
template< class InputIt, class T >
T accumulate( InputIt first, InputIt last, T init );
```

判等：

```
template< class InputIt1, class InputIt2 >
bool equal( InputIt1 first1, InputIt1 last1,
            InputIt2 first2 );
```

迭代：

```
template< class InputIt, class UnaryFunction >
UnaryFunction for_each( InputIt first, InputIt last, UnaryFunction f );
```

查找：

```
template< class InputIt, class T >
InputIt find( InputIt first, InputIt last, const T& value );
```

注意这些算法需要的最低条件，你可以提供功能更强大的迭代器作为输入。

### 输出迭代器

和输入迭代器相对的一个概念是输出迭代器，它要求的核心功能是能够对指定的元素写入数据：也就是说支持 `*i = x`，需要注意的是，这是一个只读的概念，支持写入并不代表支持读取数据，比如 `x = *i` 不一定合法。因为`*`操作符可能返回的是对象，这个对象可能并不支持拷贝或者到`x`类型的转换。输出迭代器通常用于算法的输出参数，比如：

拷贝：

```
template< class InputIt, class OutputIt >
OutputIt copy( InputIt first, InputIt last, OutputIt d_first );
```

### 前向迭代器

前向迭代器是一个支持数据多次读取的输入迭代器，如果前向迭代器是一个可变（mutable相对于只读const而言）迭代器的话，它符合输出迭代器的要求。

前向迭代器于前两个迭代器最大的区别是它支持多次读写，对于输入迭代器来说`*i == *i` 是不一定成立的，而对于前向迭代器来说却成立，输入迭代器的++操作可能使得之前保留的迭代器失效，而前向迭代器不会，也就是说你可以保存一个中间值，然后继续递增。这个特性使得它可以用于`Multipass`（表示区间可多次扫描，相对于Singlepass的单次扫描）的算法。

```
template< class ForwardIt, class T >
bool binary_search( ForwardIt first, ForwardIt last, const T& value );
```

它同时也适用于既需要输出有需要判等的情况：

```
template< class ForwardIt, class T >
void fill( ForwardIt first, ForwardIt last, const T& value );
```

### 双向迭代器

双向迭代器是一种支持双向迭代的前向迭代器，也就是说它支持`--`操作，大部分涉及到逆序相关的算法都需要使用到双向迭代器：

```
template< class BidirIt >
void reverse( BidirIt first, BidirIt last );
```

```
template< class BidirIt1, class BidirIt2 >
BidirIt2 copy_backward( BidirIt1 first, BidirIt1 last, BidirIt2 d_last );
```

### 随机迭代器

最后这个迭代器，是最接近指针概念的迭代器。它是一种能在常量时间内指向任意元素的双向迭代器。它要求迭代器支持算数运行，也就是 `i + n`，`b - a` 这一类的操作。这种迭代器要求比较高，能够实现的算法的效率也通常比较高。

打乱数据：

```
template< class RandomIt, class URBG >
void shuffle( RandomIt first, RandomIt last, URBG&& g );
```

排序：

```
template< class RandomIt >
void sort( RandomIt first, RandomIt last );
```

## tag

# 迭代器适配器
