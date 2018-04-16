---
title: C++轮子——STL算法和迭代器
date: 2018-04-13 15:44:39
tags:
 - C/CPP
---

STL的六大组件中最主要的是容器和算法这两个，一个泛化数据的存储，一个泛化数据的操作。前面两篇文章我们简单的介绍了STL中的容器，这篇文章将会介绍STL算法以及粘合容器和算法的迭代器。STL是基于模板实现，容器基于模板类，而算法基于模板函数。在具体介绍算法和迭代器之前，我们先简单的回顾一下模板函数的语法。

<!--more-->

# 模板函数

模板函数的语法其实很简单，只要把正常的函数的参数类型或者返回值类型都参数化就可以了。比如选择两个数中的最大值，我们可以使用`std::max`：

```C++
template< class T >
const T& max( const T& a, const T& b );
```

这个模板函数中两个参数的类型和返回值的类型都参数化了。

## 自动类型推导

模板函数有一个非常重要的特性是——它支持类型的自动推导。我们实例化一个模板类的时候需要手动指定模板参数类型：

```C++
std::vector<int> iv;
```

但是我们实例化一个模板函数却通常不需要，因为参数可以自动推导【1】：

```C++
std::max(1, 999);
```

当然有些情况下自动推导会失败（下面这个例子中，两个实参的类型不一样，自动推导有歧义），这个时候我们可以显式指定模板函数的参数类型：

```C++
std::max<double>(1, 2.0);
```

## 简化模板类的创建

模板函数参数的自动推导在使用上非常方便，这一点被广泛的用于模板类的工厂方法的实现（广义上任何用于创建类的实例的方法都可以称为工厂方法）。比如我们要构造一个`std::pair`，有两种方式。

第一，显式指定模板类参数：

```C++
std::pair<int, int> point(1, 2);
```

第二，使用模板函数自动推导：

```C++
std::make_pair(1, 2);
```

很显然后面这种方式用起来会比前面的方式舒服一些。标准库中存在大量的这一类型的工厂方法，比如：`std::make_tuple`，`std::make_excpetion_ptr`，`std::make_shared<T>`等等。

# 数据和操作

容器和算法的关系，实际上对应着数据和操作的关系。 计算机领域有一个非常著名的公式：

> 程序 = 数据结构 + 算法

换句话说，我们可以认为【2】：

> 程序 = 数据 + 操作

在STL中，容器抽象了数据的存储，算法抽象了数据的操作。当然这个说法其实并不准确，因为数据的操作还有一部分（比如插入删除等）直接放到了容器的内部作为成员函数而存在。操作应该实现为成员函数还是算法，主要取决于这个操作是否和数据存储相关（还有可能和效率相关）。理论上说和存储方式无关的操作都可以实现为算法【3】。

# 算法和数据类型、数据存储方式无关

下面这段算法的定义来自《算法导论》

> An algorithm is a sequence of computational steps thats transform the input
> into the output.

翻译成中文是说

> 算法是把输入变成输出的一系列计算步骤

这段定义有几个隐含的点值得讨论：

## 1. 算法和输入数据的类型无关

当我们描述快速排序算法的时候，我们其实在描述排序的步骤，至于数据类型是`int`还是`double`和算法本身其实没有关系。这个概念在C语言中比较不好表达，通常需要通过`void*`加上函数指针来实现。比如C语言中的排序算法定义如下：

```C
void qsort( void *ptr, size_t count, size_t size,
            int (*comp)(const void *, const void *) );
```

C++中因为模板函数的存在，参数类型可以被参数化，所以这个问题解决起来就方便很多，C++中的`std::sort`定义如下：

```C++
template< class RandomIt >
void sort( RandomIt first, RandomIt last );
```

现在，`std::sort`只需要实现算法逻辑，不需要考虑数据类型。 这个定义显然比C的定义要简单很多也清晰很多

## 2. 算法和输入的数据是如何存储无关【4】

我们描述一个算法，说的是数据的操作步骤，至于如何完成这些操作，实际上并没有规定。在C语言中，操作如何完成和数据如何存储有很大的关系，比如我们要实现指针的递增操作，我们的数据必须是连续存储的。但是在C++中，通过操作符的重载可以让数据的操作和数据如何存储的解耦开来。比如在C语言中线性查找算法的一个典型例子：查找一个字符串中的指定字符的函数`strchr`定义如下：

```C
char *strchr( const char *str, int ch );
```

而在C++中，查找线性查找算法`std::find`定义如下：

```C++
template< class InputIt, class T >
InputIt find( InputIt first, InputIt last, const T& value );
```

`strchar`要求数据必须是连续存储，而且必须以`\0`结尾，而`std::find`却没有这个要求，你可以用它来查找链表中的数据。解耦数据的存储和算法实现的关键在于输入数据的泛化，而这个泛化的关键在于迭代器组成的区间。

# 区间

前面提到算法是把输入变成输出的计算步骤，所以要写一个范型算法，首先要解决的问题是如何表达输入。一个范型算法的输入通常是由两个迭代器组成的左闭右开的区间表示的，C语言中则通常是一个首地址+长度这种方式。使用半开闭区间的方式有下面这些好处：

- 空集的概念很好表示，首尾相同即可 [beg, beg)
- 比较容易返回错误值，数据查找，如果没有找到，我们不需要返回一个特殊值，直接返回`end`，就可以来，因为`end`不在区间内部，返回它很好的表达没有找到这个概念。
- 比较容易表达迭代终止条件这个概念，`beg == end`即可表示迭代终止，这对于迭代器来说是很重要的，因为它不需要支持算数操作符，只需要支持判等符即可。

大多算法如果有输出，基本上也都是返回一个迭代器，如果成功，返回区间内的值，如果失败返回end。

## 获取区间

在C++11之前，我们通常使用容器提供的`begin`和`end`成员来获取区间：

```C++
std::vector<int> ia = {1, 3, 4, 2, 8};
std::find(ia.begin(), ia.end(), 3);
```

在C++11提供来范型函数`std::begin`，`std::end`，来获取区间（STL刚开始就是用这两个函数，只是加入标准库的过程中被去掉，现在又加回来了）。

```C++
char ia[] = {1, 3, 4, 2, 8};
std::find(std::begin(ia), std::end(ia), 3);
```

PS：数组大小的自动推导也是一个非常有意思的点，有兴趣的可以自行了解一下。

## 简化

其实每次都要调用`std::begin`、`std::end`，非常无聊，一不小心还可能输错。`boost::range`库可以帮我们简化这个过程：

```C++
std::vector<int> ia = {1, 3, 4, 2, 8};
boost::find(ia);
```

# 迭代器

前一节提到算法的输入通常是一个区间，而这个区间由两个迭代器组成。迭代器是指针这个概念的泛化，和其他所有的泛型定义一样，它实际上是一个[Concept](http://devdocs.io/cpp/concept/iterator)，基本定义如下：

> The Iterator concept describes types that can be used to identify and traverse the elements of a container.

也就是说它主要提供两个功能，标识元素以及遍历容器。对于一个普通的泛型算法来说，遍历功能表现在它的参数上面，而标识功能表现在它的返回值。这两个功能的存在意味着所有的迭代器都支持下面两个操作：

- `*i`解引用，因为迭代器支持标识元素，解引用可能是用于取值也可能是用于赋值，具体看后文迭代器的分类。

- `++i`，因为迭代器支持遍历。


迭代器作为容器（这里说的容器是广义上的容器，C数组也算是容器之一）和算法之间的桥梁而存在。但是算法的输入通常都不是迭代器这个Concept，而是这个Concept的5个Refinement。

## 迭代器的分类

迭代器虽然是指针的泛化版本，但是大部分的算法实际要求的特性会比指针少很多，为了尽可能的提升算法的适用性，迭代器的功能被拆分成5个不同的Concept：输入迭代器，输出迭代器，前向迭代器，双向迭代器，随机迭代器。

### 输入迭代器

输入迭代器提供两个主要的功能：

- 取值
- 判等

这可以说是非常简单的功能（当然还要加上迭代器自身支持的操作，见前文），但是仅仅基于这两个功能我们就可以写出很多有用的算法：

比如求和：

```C++
template< class InputIt, class T >
T accumulate( InputIt first, InputIt last, T init );
```

判等：

```C++
template< class InputIt1, class InputIt2 >
bool equal( InputIt1 first1, InputIt1 last1,
            InputIt2 first2 );
```

迭代：

```C++
template< class InputIt, class UnaryFunction >
UnaryFunction for_each( InputIt first, InputIt last, UnaryFunction f );
```

查找：

```C++
template< class InputIt, class T >
InputIt find( InputIt first, InputIt last, const T& value );
```

### 输出迭代器

和输入迭代器相对的一个概念是输出迭代器，它要求的核心功能是能够对指定的元素写入数据：也就是说支持 `*i = x`。需要注意的是，这是一个只写的概念，支持写入并不代表支持读取数据，比如 `x = *i` 不一定合法。因为`*`操作符可能返回的是对象（这种对象通常称为代理），而这个对象可能并不支持拷贝构造或者到`x`类型的转换。

输出迭代器通常用于算法的输出参数，比如：

拷贝：

```C++
template< class InputIt, class OutputIt >
OutputIt copy( InputIt first, InputIt last, OutputIt d_first );
```

### 前向迭代器

前向迭代器是一个支持数据多次读取的输入迭代器，如果前向迭代器是一个可变（mutable相对于只读const而言）迭代器的话，它符合输出迭代器的要求。

前向迭代器和前两个迭代器最大的区别是它支持多次读写，对于输入迭代器来说`*i == *i` 是不一定成立的（多次读取同一个迭代器可能读到不同的值，比如典型的`std::istream_iterator`），而对于前向迭代器来说却成立。这个特性使得它可以用于`Multipass`（表示区间可多次扫描，相对于Singlepass的单次扫描）的算法。

```C++
template< class ForwardIt, class T >
bool binary_search( ForwardIt first, ForwardIt last, const T& value );
```

它同时也适用于既需要输出又需要判等的情况（输出迭代器不支持判等）：

```C++
template< class ForwardIt, class T >
void fill( ForwardIt first, ForwardIt last, const T& value );
```

### 双向迭代器

双向迭代器是一种支持双向迭代的前向迭代器，也就是说它支持`--`操作，大部分涉及到逆序相关的算法都需要使用到双向迭代器：

```C++
template< class BidirIt >
void reverse( BidirIt first, BidirIt last );
```

```C++
template< class BidirIt1, class BidirIt2 >
BidirIt2 copy_backward( BidirIt1 first, BidirIt1 last, BidirIt2 d_last );
```

### 随机迭代器

最后这个迭代器，是最接近指针概念的迭代器。它是一种能在常量时间内指向任意元素的双向迭代器。它要求迭代器支持算数运行，也就是 `i + n`，`b - a` 这一类的操作。这种迭代器要求比较高，能够实现比较复杂的算法。

比如，打乱数据：

```C++
template< class RandomIt, class URBG >
void shuffle( RandomIt first, RandomIt last, URBG&& g );
```

又如，排序：

```C++
template< class RandomIt >
void sort( RandomIt first, RandomIt last );
```

## 层次关系图

做一个简单的总结的话，迭代器的分类大概像下面这样子。OutputIterator和ForwardIterator直接的线故意没有画上箭头，是因为他们不完全是Concept和Refinement的关系。

```
+---------------+            +----------------+
| InputIterator |            | OutputIterator |
+----------+----+            +-------+--------+
           |                         |
           |                         |
           |  +------------------+   |
           +--> ForwardIterator  +---+
              +--------+---------+
                       |
                       |
            +----------v------------+
            | BidirectionalIterator |
            +----------+------------+
                       |
                       |
            +----------v------------+
            | RandomAccessIterator  |
            +-----------------------+

```

## 区分迭代器实际类别的重要性

需要我们注意的一点是，算法对于迭代器的要求定义的是最低条件，而不是特定条件。迭代器之间其实存在一种递进关系，如果一个算法要求输入InputIterator，那么你输入Forwarditerator也是可行的。算法的实现通常会充分的利用了这一点，来最大限度的提升性能。比如STL中提供了`std::advance`这个算法用于向前移动迭代器（C
++11之后有`std::next`和对应`std::prev`，`std::next`有返回值，但是在C++17之前要求输入前向迭代器），这个迭代器要求输入输入迭代器。

```C++
template< class InputIt, class Distance >
void advance( InputIt& it, Distance n );
```

它保证最差的情况下是线性复杂度，但是如果我们输入的是随机迭代器，它可以达到常量复杂度，因为随机迭代器支持简单的算数运算可以通过`+n`的方式直接返回结果。此外如果你给定的迭代器是双向迭代器，它还支持向后移动。

解决上面的问题的其中一种方式是使用函数重载，额外提供`advance_random`，和`advance_bidirctional`两个函数，把选择权抛给用户。这种方式实现简单，但是接口复杂，而且难用。

STL在接口上并没有提供`std::advance_random`这样的接口，而是使用了统一的接口，但这意味着这个算法的内部必须判断输入的迭代器的类别（注意用词，它需要知道迭代器属于什么类别而不是迭代器到底是什么类型，比如`int*`的类型是int指针，而迭代器分类中属于随机迭代器，下文中的表述除非特殊说明，否则说的都是类别classification而不是类型type）。

传统的OO思维类型判断可能是提供一个方法返回类型，但是STL并不是这样做的，它通过另外一种更加灵活的技术——`Traits`——来实现迭代器所属类别的判断。

## std::iterator_traits

`traits`技术的一个很实用的模板技术，它通常用于特征或者说的提取。`std::iterator_traits`是标准库中提供的`traits`之一用于提取迭代器的特征（其他的还有`char_traits`，`number_traits`）。这个类通常内部至于类型定义，没有实际的数据成员和方法（有成员方法的又被称为`Policy`，当然这种分类是概念上的分类，实际上标准库中有很多`traits`类有成员方法，比如`char_traits`，`number_traits`）。

`std::iterator_traits`的定义是一个空的模板，不同的迭代器对它做特化实现静多态。

```C++
template< class Iterator>
struct iterator_traits;
```

这种实现方式我们在前面讲`std::hash`的使用也有提到过，它最大的优势在于高度的灵活性，因为它和实际需要获取特征的迭代器是解耦合的。相比于使用成员的方式，它最大的优势是可以兼容指针，你没有办法给指针加上成员，但是你可以对指针加上特化。

```C++
template< class T >
struct iterator_traits<T*>;

template< class T >
struct iterator_traits<T*>;
```

`std::hash`中使用到的特化技术是全特化技术，而这里使用到的偏特化技术【5】。

`std::iterator_traits`定义了下面五个类型成员（member type）

- `difference_type`
- `value_type`
- `pointer`
- `reference`
- `iterator_category`

比如对于指针的特化中，上面这几个类型被定义为下面的类型：

| Member type         | Definition                        |
|------------         | ----------                        |
| `difference_type`   | `std::ptrdiff_t`                  |
| `value_type`        | `T`                               |
| `pointer`           | `T*`                              |
| `reference`         | `T&`                              |
| `iterator_category` | `std::random_access_iterator_tag` |

其中最后一个类型可以用于显示指针属于随机迭代器。

## Concept在类型系统的表示方式：TAG

指针的迭代器分类被定义为：`std::random_access_iterator_tag`，这是一个很有意思的话题，因为它用类型来表示来Concept，而前面提到Concept并不是一个类型而是符合某些条件的类型的集合。因为它不是具体的类型，所以这些tag实际上只用做类别区分，并没有任何的成员。此外而Concept，Refinment的关系实际上也被表示为空类的父子继承关系：

```C++
struct input_iterator_tag { };
struct output_iterator_tag { };
struct forward_iterator_tag : public input_iterator_tag { };
struct bidirectional_iterator_tag : public forward_iterator_tag { };
struct random_access_iterator_tag : public bidirectional_iterator_tag { };
```

注意看，`forward_iterator_tag`只是继承了`input_iterator_tag`。

在做`traits`的特化的时候，我们可以用上面这些TAG来表示迭代器的实际分类。

```C++
typedef std::random_access_iterator_tag iterator_category;
```

标准库中存在一个工具类 `[std::iterator](http://devdocs.io/cpp/iterator/iterator)` 帮助我们定义上面这些类型。

## 算法是如何利用迭代器的分类信息的

前面提到，充分利用迭代器的分类信息可以提升算法的性能，而算法利用分类信息的方式通常是函数重载，我们回头来看`std::advance`的实现方式【6】：

```C++
template <class _InputIter>
inline _LIBCPP_INLINE_VISIBILITY _LIBCPP_CONSTEXPR_AFTER_CXX14
void __advance(_InputIter& __i,
             typename iterator_traits<_InputIter>::difference_type __n, input_iterator_tag)
{
    for (; __n > 0; --__n)
        ++__i;
}

template <class _BiDirIter>
inline _LIBCPP_INLINE_VISIBILITY _LIBCPP_CONSTEXPR_AFTER_CXX14
void __advance(_BiDirIter& __i,
             typename iterator_traits<_BiDirIter>::difference_type __n, bidirectional_iterator_tag)
{
    if (__n >= 0)
        for (; __n > 0; --__n)
            ++__i;
    else
        for (; __n < 0; ++__n)
            --__i;
}

template <class _RandIter>
inline _LIBCPP_INLINE_VISIBILITY _LIBCPP_CONSTEXPR_AFTER_CXX14
void __advance(_RandIter& __i,
             typename iterator_traits<_RandIter>::difference_type __n, random_access_iterator_tag)
{
   __i += __n;
}

template <class _InputIter>
inline _LIBCPP_INLINE_VISIBILITY _LIBCPP_CONSTEXPR_AFTER_CXX14
void advance(_InputIter& __i,
             typename iterator_traits<_InputIter>::difference_type __n)
{
    __advance(__i, __n, typename iterator_traits<_InputIter>::iterator_category());
}
```

实现的关键在于在内部构造一个空对象来完成函数分派，这种技术我们在前面讲[std::array](/2018/02/25/stl-part-two-vector/#%E9%9D%9E%E7%B1%BB%E5%9E%8B%E5%8F%82%E6%95%B0)的时候顺带提到过一次。

## 常见数据结构的迭代器

C++中存在大量的数据结构，包括容器，数组，字符串等，这些常用的数据结构都提供迭代器给我们使用，这里简单的罗列一下他们提供的迭代器类型：

| 数据结构       | 迭代器类别 |
| ------------   | ---------- |
| `C 数组`       | `随机`     |
| `字符串`       | `随机`     |
| `vector`       | `随机`     |
| `deque`        | `随机`     |
| `list`         | `双向`     |
| `map`          | `双向`     |
| `set`          | `双向`     |
| `forward_list` | `前向`     |

## 算法和迭代器使用上需要注意的点

### 区间合法性

算法的输入通常是一个迭代器组成的区间 [first，last），遍历的时候从`first`开始遍历直到`first == last` 为止，如果 first 用于到不了 last，会形成死循环。

### 迭代器失效问题

迭代器是指针的泛化版本，它实际上遗传了指针的一些问题：比如野指针。这个问题在容器插入和删除的时候非常常见，比如在`std::vector`尾部插入数据，可能导致底层数组的重新分配，它会导致之前的迭代器失效，类似野指针。

```C++
std::vector<int> iv = {1, 2};
auto beg = std::begin(iv);
iv.push_back(3);

*beg = 4;   // 可能会崩溃
```

### 越界

和指针一样，迭代器通常不检查越界问题，所以如果你传入的区间比实际的区间要大，这将会导致崩溃。在一点在隐式区间中很容易出现：

```C++
int a[] = {1, 3, 5, 7, 9};
std::vector<int> iv;
std::copy(std::begin(a), std::end(a), std::begin(iv));
```

上面这段代码非法，因为`std::copy()`第三个参数包含一隐含的大小为`std::end(a) -
std::begin(a)`的区间，而`iv`是一个空数组，显然不存在这样的区间。

## 安全模式

解决上面说到的这些问题，一种比较简单的方式是在Debug版本中使用`Safe
Mode`，不同的编译器提供来不同的方式开启`Safe Mode`，比如GCC中可以使用`-D_GLIBCXX_DEBUG`来开启，MSVC在Debug模式下会默认执行检测。

因为迭代器是一个Concept，而不是一个类，所以STL的实现者可以把迭代器定义为一个具有检查功能的代理类从而提供检查功能。

## 迭代器适配器

解决迭代越界的另外一种方式是使用，迭代器适配器，比如说上面提供的问题可以通过下面这种方式解决：

```C++
int a[] = {1, 3, 5, 7, 9};
std::vector<int> iv;
std::copy(std::begin(a), std::end(a), std::back_inserter(iv));
```

`std::back_inserter`实际上是一个工厂方法（参考前面`std::make_pair`），它帮我们创建一个`std::back_inserter_iterator`。而`std::back_inserter_iterator`是一个迭代器适配器。

前面讲序列容器的时候有提到容器适配器，它们都是适配器，但是适配的方向不同。容器适配器把容器的接口适配成其他的接口，而迭代器适配器把别的接口适配成迭代器的接口。最本质的区别就是，容器适配器不是容器，而迭代器适配器是迭代器。

STL中提供了大量的迭代器适配器，主要的包括下面几类：

### 插入类

这三个都是输出迭代器：

- `back_inserter_iterator`，用 `push_back()` 适配
- `front_insert_iterator`，用 `push_front()` 适配
- `insert_iterator`，用 `insert()` 适配

### 输入输出

这些迭代器负责把输入输出库和迭代器融合到一起，主要有`istream_iterator`，`ostream_iterator`，`istreambuf_iterator`，`ostreambuf_iterator`。

```
std::istringstream str("0.1 0.2 0.3 0.4");
std::partial_sum(std::istream_iterator<double>(str),
                 std::istream_iterator<double>(),
                 std::ostream_iterator<double>(std::cout, " "));
```

输出：

> 0.1 0.3 0.6 1

PS: 对于字符串的处理，`std::istringstream`配合算法可以达到非常好的效果。

### 逆序迭代器

`std::reverse_iterator`可以把正向的遍历变成逆序遍历，比如查找，默认查找第一个，但是如果我使用`std::reverse_iterator`适配一下，我们可以轻易的实现查找最后一个元素的功能。

逆序迭代器在实现上有一个难点就是区间的合法性，算法的区间，大部分都是定义为`[begin, end)`，这种形式，读取`begin`合法而读取`end`不合法。但是区间如果反过来，`(begin -1, end -1 ]`这个区间却不一定合法，因为`begin - 1`通常不是一个合法的迭代器。为了解决这个问题，逆序迭代器中的元素物理位置和逻辑位置有一个元素的偏差。物理上我们依旧使用`[begin, end)`这个区间，逻辑上对`end` 的操作存取的是`end - 1` 这个位置。物理位置到逻辑位置的转换可以通过 `base` 成员返回。【7】。

---

【1】：理解模板参数的自动推导是理解C++11中auto的关键，关于这一点《Effctive Modern C++》第一章有详细的介绍。

【2】：这个是个人理解，和上面的等式并不能划等号

【3】：算法是普通的成员函数，一个操作应该实现为普通的函数还是成员函数可以参考《Effctive C++ 第三版》条款23。

【4】： 算法和数据的存储方式无关这一点，是STL中容器和算法库分离的很重的一个原因，而这种分离可以极大的提升算法的适用范围，降低代码重复性。比如Java中的`ArrayList`和`LinkedList`都实现了`indexOf`方法来实现查找的功能，而C++中只实现了一个`std::find`它可以通吃`c array`，`std::array`，`std::vector`，`std::deque`，`std::list`，`std::string`等，这其实也是范型算法的魅力所在。

【5】：全特化和偏特化的区别请参考《C++模板》一书

【6】：这是 libcxx 目前的实现 2018-04-16

【7】：更详细的细节可以参考[标准](http://devdocs.io/cpp/iterator/reverse_iterator)，或者《Effctive STL》一书
