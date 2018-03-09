---
title: C++轮子 —— STL 容器
date: 2018-02-25 17:23:57
tags:
  - C/CPP
  - C++轮子
---

STL中大家最耳熟能详的可能就是容器，这篇文章中将会重点介绍STL中的各种容器和容器适配器。主要内容包括

- std::vector
- std::deque
- std::queue
- std::stack
- std::list
- std::set
- std::map

<!--more-->

# std::vector

提到STL，大部分人的第一反应是容易，而提到容器大部分人首先想到的是`vector`。斯特劳斯特卢普的观点来说，`vector`是所有的容器中的首先，如果你不清楚应该使用哪个容器，那就选`vector`吧（当然，你不应该不清楚选哪个容器，合格是程序员对自己写的代码应该要了如指掌）。

`vector`的使用非常简单，下面是一个简单的例子。

```c++
#include <vector>                               // 1

int main(int argc, char* argv[]) {
    std::vector<int> ages = { 1, 2, 3, 4 };     // 2
    return 0;
}
```

## 头文件

`// 1`中引入了`std::vector`的头文件，需要注意的是所有C++标准库的头文件都是没有`.h`结尾的。这么做是为了区分，C标准库的头文件和C++标准库的头文件。比如最具代表性的：

```c++
#include <string.h>     // C 标准库头文件，包含 strlen，memset 等函数
#include <string>       // C++ 标准库头文件，包含 std::string 类
```

此外对于所有C标准库头文件，如果你是在C++项目中引用，你应该使用`#include <cxxx>`这种方式而不是`#include <xxx.h>`这种形式。也就是说我们应该使用`#include <cstring>`而不是`#include <string.h>`

## std::vector 还是 vector

我见过很多的人（包括很多书）的习惯是在源文件头部写上`using namespace std;`然后在代码中使用`vector<int>`，而不是直接使用`std::vector<int>`。

我个人的习惯是直接使用`std::vector<int>`，因为`namespace`对我来说是一个模块，写明了`std::`有更强的模块内聚表达力，而且也不太容易出现名字碰撞。

## 初始化

`// 2`在构造`vector`的时候直接给了初始值，这是`C++11`的特性，在`C++11`之前不能这样写，有一种大致等同的写法如下：

```c++
int initilizer[4] = { 1, 2, 3, 4 };
std::vector<int> ages(initilizer, initilizer + 4);
```

`std::vector<int> ages = { 1, 2, 3, 4 }`这种写法实际上从语法分析上来说是分成下面几个步骤的：

1. `{ 1, 2, 3, 4 }` 被编译器构造成一个临时变量`std::initializer_list<int>`，然后使用临时变量构造一个临时变量 `std::vector<int>`，然后再用 `std::vector`的拷贝构造函数构造最终的`ages`

```c++
std::initializer_list<int> initilizer;
std::vector<int> tmp(initilizer);
std::vector<int> ags(tmp);
```

当然上面的分析只是语法上的分析，绝大部分编译器都可以优化掉`tmp`，而且因为`{1, 2, 3, 4}`转换成`std::initializer_list`是编译器在编译器完成的事情，所以其实效率比我们想象中要高一些。

## std::vector<bool>

`std::vector`有一个特化版本`std::vector<bool>`，用于实现`dynamic bitset`，需要注意的是，这个特化版本并不是容器，它的迭代器无法很好的适配`STL`中的所有算法。它的存在是为了节省空间，它的每一个元素只占用一位而不是一个字节。为了实现这种优化，`operator[]`返回的是一个代理类，你没有办法取单个元素的地址。通常的建议是，如果你不需要动态的`bitset`，你可以使用`std::bitset`，如果你需要`dynamic bitset`你可以考虑使用`deque<bool>`替代。

## `push_back` vs `emplace_back`

C++11在容器尾部添加一个元素调用的函数是`push_back`，它在`libcxx`中的实现如下：

```c++
template <class _Tp, class _Allocator>
inline _LIBCPP_INLINE_VISIBILITY
void
vector<_Tp, _Allocator>::push_back(const_reference __x)
{
    if (this->__end_ != this->__end_cap())
    {
        __RAII_IncreaseAnnotator __annotator(*this);
        __alloc_traits::construct(this->__alloc(),
                                  _VSTD::__to_raw_pointer(this->__end_), __x);
        __annotator.__done();
        ++this->__end_;
    }
    else
        __push_back_slow_path(__x);
}
```

这里存在两次元素的构造，一次是 __x 参数的构造，一次是容器内部原始的拷贝构造。也就是说使用拷贝构造在末尾构造一个新的元素。`emplace_back`是C++11为减少其中一次拷贝而引入的新的接口，在`libcxx`中的实现如下

```c++
template <class _Tp, class _Allocator>
template <class... _Args>
inline
#if _LIBCPP_STD_VER > 14
typename vector<_Tp, _Allocator>::reference
#else
void
#endif
vector<_Tp, _Allocator>::emplace_back(_Args&&... __args)
{
    if (this->__end_ < this->__end_cap())
    {
        __RAII_IncreaseAnnotator __annotator(*this);
        __alloc_traits::construct(this->__alloc(),
                                  _VSTD::__to_raw_pointer(this->__end_),
                                  _VSTD::forward<_Args>(__args)...);
        __annotator.__done();
        ++this->__end_;
    }
    else
        __emplace_back_slow_path(_VSTD::forward<_Args>(__args)...);
#if _LIBCPP_STD_VER > 14
    return this->back();
#endif
}
```

## back() 和 pop_back()

std::vector内部的数据使用连续的空间存储，除了在尾部插入和删除之外都会需要涉及到其他元素的挪动（空间不足的时候在尾部插入也会需要挪动元素）。std::vector 提供了两个接口用于删除尾部数据，

```c++
reference back();
void pop_back();
```

理想状态下，我们可以用一个接口完成

```c++
value_type pop_back();
```

之所以分开成两个接口是为了保证异常安全，如果让`pop_back`返回尾部数据就必然涉及到尾部数据的拷贝，而这个拷贝可能抛出异常导致数据的丢失。

## 自动增长

`std::vector` 会在内存不够的时候自动增长空间，这是相对于C数组来说最大的一个优势。那么空间不够的时候怎么增长呢？答案是不知道。之所以特地强调这一点是为了说明C++标准一个非常重要的特点，规定结果不规定实现，甚至不规定结果。很多人因此抨击C++，带着来自其他语言的优越感嘲笑着为不确定性而焦头烂额的C++工程师。标准之所以这样规定，很大程度上承袭自C语言，C语言标准在很多地方不给出确定的结果是为了给编译器最大的选择，从而达到性能的最优化，毕竟那个时代性能真的很重要。

实际上标准规定`push_back`是`Amortized constant`，但并没有规定应该怎么实现，所以最准确的答案是不知道，虽然如果要实现`Amortized constant`，必须分配 2 倍以上的空间。

这里之所有特地强调这个点，是希望大家重视C++代码的可移植性，做到这一点的关键就是以来行为而不依赖实现。C++作为系统编程语言，几乎可以在任何平台上跑，它的可移植性实际上远高于Java语言。

## 缩减 std::vector

`std::vector`会在空间不够的时候自动分配空间，但是它并不会在空间冗余的时候自动释放空间。如果你使用`C++11`之后的版本，你可以使用`std::vector::shrink_to_fit`来回收空间，否则你需要像下面这个缩减空间。

```c++
std::vector<int> ages;
std::vector<int>(ages.begin(), ages.end()).swap(ages)
```

这种用法叫做`copy and swap`在拷贝构造函数的实现中用的也很多。这个地方需要特别注意的是临时变量和实际变量位置不能写反`ages.swap(std::vector<int>(ages.begin(), ages.end()))`是语法错误，因为`std::vector::swap`的原型如下：

```c++
void swap( vector& other );
```

临时变量（前面哪个匿名对象）是右值，无法绑定到一个左值引用上面。

## 兼容 C 数组

C++很重要的一个特性就是兼容C语言，C的接口中，如果需要传入一个数组，通常的方式s是传入一个起始地址加上一个长度，如下：

```c++
void* memset( void* dest, int ch, std::size_t count );
```

如果你现在有一个`std::vector`，现在需要把它传递给C，接口你可以调用`std::vector::data`这个成员变量获取底层的内存空间的首地址。`std::vector`和其他的容器一个非常重要的区别就是它保证了底层的内存空间的连续性，也就是说，它保证了内存空间和C数组的兼容性，能用C数组的地方都可以使用`std::vector`，而且它还能保证内存空间的自动管理。

## std::vector 的内存模型

我们来看下面这段代码：

```
std::vector small(100);
std::vector large(1000);
```

那么 `sizeof(small)` 和 `sizeof(large)` 哪个大呢？答案是一样大，要解答这个问题我们需要了解 `std::vector` 的内存模型。`std::vector`的实现的内存模型并不完全一样，但是基本上都大同小异，类似下面这种结构。

```c++
            stack
        +------------+
        |  begin     +----------+
        +------------+          |
        |  end       +-------------------------------+
        +------------+          |                    |
+-------+  cap       |          v                    v
|       +------------+          +--------------------+----------+
|       |   ......   |          |                    |          |   heap
|       +------------+          +--------------------+----------+
|                                                               ^
+---------------------------------------------------------------+
```

从上面的图中我们可以看出`small`和`large`真正的差别其实在`heap`不在`stack`，所以说`sizeof(small) == sizeof(large)`。

## 返回一个 std::vector 的开销到底有多大

大概是受C语言的影响，很多人觉得返回一个`std::vector`的开销很大，很多人为了能够避免这部分开销，选择使用输入参数的方式返回函数的返回值。

```c++
void foo(std::vector<int>& out);
```

这种写法和下面的写法到底有多大的开销上的区别呢？

```
std::vector foo();
```

在我看来其实差别并不大，原因主要有两个：

1. 因为`RVO`和`NRVO`的存在，第一种写法和第二种写法很有可能在最后生成的代码上是一致的。

2.  及时`foo`的实现导致没有办法做`RVO`，在`C++11`中返回一个`std::vector`的开销实际上也不大，以为返回值都是以`move`而不是以`copy`的方式返回的，也就是说真正拷贝的部分只是`stack`上的那一部分，`heap`那一部分是不需要拷贝的。

所以大部分情况下，你都可以放心的返回一个 std::vector，因为相对下引用参数的写法，直接用返回值会有更好的代码可读性。

```c++
std::vector<int> param_out;
foo(param_out);

// 远没有下面的写法直观

std::vector out = foo();
```

[1】C++标准只规定了结果，并不规定如何实现，不同的C++编译器对于`std::vector`的实现可能完全不一样，这里看的源码是来自llvm的`libcxx`实现。
