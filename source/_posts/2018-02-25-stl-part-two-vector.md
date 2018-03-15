---
title: C++轮子 —— STL 序列容器
date: 2018-02-25 17:23:57
tags:
  - C/CPP
  - C++轮子
---

STL中大家最耳熟能详的可能就是容器，容器大致可以分为两类，序列型容器（SequenceContainer）和关联型容器（AssociativeContainer）这篇文章中将会重点介绍STL中的各种序列型容器和相关的容器适配器。主要内容包括

- std::vector
- std::array
- std::deque
- std::queue
- std::stack
- std::list
- std::forward_list

<!--more-->

# std::vector

提到STL，大部分人的第一反应是容器，而提到容器大部分人首先想到的是`std::vector`。斯特劳斯特卢普的观点来说，`std::vector`是所有的容器中的首先，如果你不清楚应该使用哪个容器，那就选`std::vector`吧（当然，你不应该不清楚选哪个容器，合格是程序员对自己写的代码应该要了如指掌）。

`std::vector`的使用非常简单，下面是一个简单的例子。

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

`// 2`在构造`std::vector`的时候直接给了初始值，这是`C++11`的特性，在`C++11`之前不能这样写，有一种大致等同的写法如下：

```c++
int initilizer[4] = { 1, 2, 3, 4 };
std::vector<int> ages(initilizer, initilizer + 4);
```

`std::vector<int> ages = { 1, 2, 3, 4 }`这种写法实际上从语法分析上来说是分成下面几个步骤的：

1. `{ 1, 2, 3, 4 }` 被编译器构造成一个临时变量`std::initializer_list<int>`，然后使用临时变量构造一个临时变量 `std::vector<int>`，然后再用 `std::vector<int>`的拷贝构造函数构造最终的`ages`

```c++
std::initializer_list<int> initilizer;
std::vector<int> tmp(initilizer);
std::vector<int> ags(tmp);
```

当然上面的分析只是语法上的分析，绝大部分编译器都可以优化掉`tmp`，而且因为`{1, 2, 3, 4}`转换成`std::initializer_list`是编译器在编译器完成的事情，所以其实效率比我们想象中要高一些。

## std::vector<bool>

`std::vector`有一个特化版本`std::vector<bool>`，用于实现`dynamic bitset`，需要注意的是，这个特化版本并不是容器，它的迭代器无法很好的适配`STL`中的所有算法。它的存在是为了节省空间，它的每一个元素只占用一位而不是一个字节。为了实现这种优化，`operator[]`返回的是一个代理类，你没有办法取单个元素的地址。通常的建议是，如果你不需要动态的`bitset`，你可以使用`std::bitset`，如果你需要`dynamic bitset`你可以考虑使用`std::deque<bool>`替代。

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

所以大部分情况下，你都可以放心的返回一个`std::vector`，因为相对下引用参数的写法，直接用返回值会有更好的代码可读性。

```c++
std::vector<int> param_out;
foo(param_out);

// 远没有下面的写法直观

std::vector out = foo();
```

# std::array

`std::vector`会自动管理使用到的内存，这是一个非常重要的特性，但是如果你的数据的大小是已知而且固定的，这个特性对于你来说是不必要的开销。因为前面提到`std::vector`的数据实际上放到heap上面，访问需要额外的解引用，而且它可能内部有内存空闲，空间有浪费。这种情况下你可以考虑使用`std::array`来替换`std::vector`。

```c++
enum LogLevel { kTrace, kInfo, kDebug, kWarning, kError };

void PrintLog(LogLevel level, const std::string& msg) {
    static const std::array<std::string, 5> kTags = {"trace", "info", "debug",
                                                     "waring", "error"};

    std::cout << "[ " << kTags[level] << " ] " << msg << std::endl;
}
```

## 非类型参数

上面的定义有一个很有意思的地方是，第二个参数是一个数值而不是一个类型，C++模板参数分为类型参数和非类型模板参数，其中后者可以使用整型、指针和引用。指针和引用我没怎么见过，整型出现的频率倒是比较高，比如上面的例子其实可以改写成下面这个样子：

```c++
enum LogLevel { kTrace, kInfo, kDebug, kWarning, kError };

template <LogLevel LEVEL>
struct LogTag {};

template <LogLevel LEVEL>
void PrintLog(const std::string& msg) {
    PrintLog(LogTag<LEVEL>(), msg);
}

void PrintLog(LogTag<kTrace>, const std::string& msg) {
    std::cout << "[ trace ] " << msg << std::endl;
}

void PrintLog(LogTag<kInfo>, const std::string& msg) {
    std::cout << "[ info ] " << msg << std::endl;
}

void PrintLog(LogTag<kDebug>, const std::string& msg) {
    std::cout << "[ debug ] " << msg << std::endl;
}

void PrintLog(LogTag<kWarning>, const std::string& msg) {
    std::cout << "[ warning ] " << msg << std::endl;
}

void PrintLog(LogTag<kError>, const std::string& msg) {
    std::cout << "[ error ] " << msg << std::endl;
}
```

这种写法的效率会比使用vector的效率要高，因为LogLevel在编译器期间就已经确定，所以在编译期间就已经确定需要调用哪一个函数，不需要做一次额外的数组访问。

## 为什么不直接使用C数组

`std::array`实际上是一个容器，它提供来迭代器可以很方便的遍历元素，它可以用过 size() 方法返回数组的大小，而且它是zero cost abstraction的绝佳体现，它的开销实际上并不比C数组要大，但是却提供来大量的方便易用的接口，可以和STL很好的整合在一起，所以如果你使用C++，你基本上可以考虑告别C数组了，变长数组你可以使用vector，定长数组可以使用`std::array`。

# std::deque

前文提到，`std::vector`的内存空间是连续的，在头部插入数据需要移动所有数据是O(N)级别的操作，因为开销过于巨大，`std::vector`并没有提供在头部插入和删除的接口。如果我们真的有这样的需求，我们可以选择使用`std::deque`。它支持在头部和尾部以O（1）的开销插入和删除数据，同时可以在O（1）时间内访问任意元素。

## `push_front`、`front`、`pop_front`

如果你选择使用`std::deque`而不是vector，十有八九你是为了用这三个函数，`std::deque`提供这三个函数用于在队列的头部插入和删除数据。需要注意的是下面两点：

- 这三个函数的复杂度都是O（1）
- 提供三个函数而不是两个函数是为了保证异常安全性

## 内存模型

`std::deque`是如何做到O（1）时间内访问任意元素又保证O（1）时间在头部和尾部操作数据内？这要从它的内存模型说起。

`std::deque`在逻辑上也是一个数组，只不过在物理上它的空间并不连续，它实际上由一段一段的小块儿内存拼接而成，这些小块儿的内存我们姑且叫它`buffer`，把这些`buffer`串在一起的就形成了一个逻辑上的一纬数组，用来串连这些`buffer`我们姑且称之为`map`

```
      +------+-------+-------+-------+--------+------+
      |      |       |       |       |        |      | map
      +------+---+---+----+--+----+--+----+---+------+
                 |        |       |       |
              +--v-+   +--v-+  +--v-+  +--v-+
              |    |   |    |  |    |  |    |  buffer
              |    |   |    |  |    |  |    |
              |    |   |    |  |    |  |    |
begin   +-->  +----+   |    |  |    |  +----+  <-+    end
              |    |   |    |  |    |  |    |
              |    |   |    |  |    |  |    |
              +----+   +----+  +----+  +----+
```

这个结构实际上是把一维数组变成了二维结构，本质上来说它就是通过增加一个间接层来实现的。再一次印证来那句老化，什么问题都可以通过增加间接层来解决。

## 逻辑上的数组

我们说逻辑上，`std::deque`也是一个数组，它支持取下标操作符，可以在O（1）时间内访问容器内部的任意元素。需要注意的是`std::deque`的O（1）和vector的O（1）存在常数上的差别，因为vector只需要一次解引用就可以获取元素而`std::deque`需要两次。

`std::deque`的这种逻辑和物理存储不一致的特性也从另外一个侧面反应了接口和实现直接的本质区别。编程的核心思想在于抽象，而抽象的核心在于分离接口和实现。

## 自动回收空间

和vector一样空间不足的时候会自动分配分配新的空间，大多数情况下只需要分配固定大小的buffer挂到map上，但是buffer过多的时候会导致map的重新分配。和vector不同的是`std::deque`会自动回收多余的空间，如果你对于运行时的内存要求非常严苛，而且会频繁的插入和删除数据可以考虑使用`std::deque`。

## 首尾插入和删除数据可以保证其他迭代器的合法性

`std::deque`还有一个特性就是如果你只是在头部或者尾部操作数据，你之前持有的迭代器不会失效，这一点我们会放到后面迭代器相关的文章中重点的讨论。

# std::queue

传统的数据结构课程中，提到同时操作头部和尾部，我们首先想到的应该是队列，它是一种FIFO的结构，广泛的使用在各种程序中。STL中提供来`std::queue`这个模板类来实现这一结构，但是需要特别注意的是它不是一个容器，它是容器适配器。

## 什么是容器

想要理解为什么我们说`std::queue`不是容器，我们需要理解前面我们一直没有讨论的一个问题——什么是容器？

加入你生活在OO的世界，你需要定义某一个概念，通常你会定义一个接口（这里说的接口是语法层面的接口而不是逻辑层面的接口），然后定义需要支持的函数，比如（伪代码）：

```c++
template<typename T>
class IContainer {
public:
    virtual ~IContainer() = default;

    virtual size_t size() = 0;
    virtual T at(size_t idx) = 0;
    ......
};

template<typename T>
class vector : public IContainer<T> {
};
```

在C++范型的世界里，并不存在类似这样的语法元素，定义某一个概念靠的是文本描述（C++后续的版本可能会改善这一点，但是目前最新的版本C++17没有）。这段文本描述称之为concept，它定义来范型世界里面的接口。比如说容器的concept如下：

![container](/img/posts/devdocs.io_cpp_concept_container.png)

也就是说，标准里面使用这样的文字描述来一个容器应该满足的条件，满足这些条件的就是一个容器。从这一点上来说，concept是一种鸭子模型，会飞、会游泳、会呱呱呱叫的就是一只鸭子，至于你是野鸭子还是橡皮鸭都不重要。

concept这个概念在范型编程中极其重要，用别人的范型类或者范型函数的时候，你需要了解它的参数的concept以保证你的参数输入合法（编译器没办法帮你检查这一点），自己写范型的时候，你需要考虑并标明你的模板参数所需要满足的concept以便别人可以正常的使用，这其实是接口使用者和接口实现者之间的合约，人无信不立这句话放在这里其实也好合适。

## 为什么不是容器

了解了容器的concept之后，我们就可以很清楚的知道为什么`std::queue`不是容器，因为它不满足容器的concept，比如它没有定义iterator这个成员类型，它也不提供begin、end这样的成员方法，也就谁说你没有办法它不提供迭代器，没有迭代器你就不能使用STL中的算法，你也就失去了STL中的半壁江山。

## 什么是容器适配器

我们说`std::queue`是一个容器适配器，所谓的适配器从设计模式的角度考虑就是把一个类的接口适配成另外一种接口。`std::queue`实际上就是拿着容器的接口，适配成队列的所需要的接口。默认情况下，它用来适配的容器是`std::deque`，这是为什么我们在讲完`std::deque`之后接着就讲`std::queue`的原因。我们来看一下标准库中`std::queue`的定义：

```c++
template<
    class T,
    class Container = std::deque<T>
> class queue;
```

上面这段定义有两个地方值得讨论：

1. 模板参数可以又默认值，就像我们普通的函数参数一样，在C++11之前只有模板类的模板参数可以有默认值在C++11之后，模板函数的模板参数也可以有默认值。

2. `std::queue`默认使用的容器是`std::deque`。也就是说如果你觉得不合适，你完全可以换掉它，只要你提供的类型满足 Container 这个模板参数需要的条件：

> The type of the underlying container to use to store the elements. The
> container must satisfy the requirements of SequenceContainer. Additionally,
> it must provide the following functions with the usual semantics:
>  - back()
>  - front()
>  - push_back()
>  - pop_front()

在标准库中除了`std::deque`之外`std::list`也满足这个条件。

## 例子

我们用一个例子结束这一部分的讨论

```c++
#include <queue>
#include <list>
#include <cassert>

int main(int argc, char *argv[])
{
    std::queue<int, std::list<int>> numbers;

    numbers.push(1);
    numbers.push(2);
    numbers.push(3);

    assert(numbers.front() == 1);
    assert(numbers.back() == 3);

    numbers.pop();
    assert(numbers.front() == 2);

    return 0;
}
```

请注意上面`std::queue`的定义，我提供了第二个参数，也请注意前面的写法是`std::list<int>`而不是`std::list`因为list不是类型，list<int>才是。

# std::stack

说完FIFO的队列，我们顺便说一下FILO的栈，在STL中提供了`std::stack`用来实现栈的功能，它和`std::queue`一样是容器适配器而不是容器，而且它同样默认使用`std::deque`作为默认的容器，和`std::queue`不同的是，`std::stack`只需要操作容器的尾部，所以你可以用vector当作来适配`std::stack`。

`std::stack`的接口比较直观这里不再赘述，有需要的同学可以自行查看devdocs或者cppreference上面的文档。

# std::list

STL中提供了`std::list`表示链表，通常它的实现是双链表（它支持双向迭代），如果你的代码中需要使用到链表结构可以选择用它做为容器，虽然它的适用场景可能会比我们想象中要低很多。

## std::vector vs std::list

传统的数据结构的教程中，`list`通常都是伴随着`array`而来，通常书上会告诉你

> `list`中元素的插入和删除比`array`要快，如果你频繁使用插入和删除你应该使用`list`而不是`array`

这个说法在学术上是可以认为是正确的，但是实际上大部分情况下，上面的说法是不靠谱的。绝大部分情况下，`std::vector`的效率都会比`std::list`要高，原因主要有下面几点：

1. 找到插入点，`std::list`需要O（N）的时间，而vector只需要O（1）的时间。
2. `std::vector`的数据是集中存储的，而`std::list`的数据是离散存储的，这意味着vector的cache命中率会比`std::list`的cache命中率要高，内存的读写效率可能会比`std::list`要高。
3. `std::list`存储一个数据需要两个指针（双链表）的额外空间，`std::vector`不需要，所以`std::vector`的内存内存使用效率会高于`std::list`。
4. `std::vector`的数据是连续的，可以使用二分查找，快速查找等算法，`std::list`不行，所以`std::vector`的查找效率可能会高于`std::list`。

所以大部分情况下你实际需要的可能都是vector而不是`std::list`，即使你伴随着数据的删除和插入。那么什么时候应该选用`std::list`呢？

## 什么时候考虑用std::list

1. 容器里面的元素比较大，这种情况下，两个指针的额外开销基本上可以接受，而且如果元素本身比较大，它自身cache的命中率会高。

2. 容器的原始特别多，而且插入删除比较频繁（而且很多在头部插入，如果都是在头部插入可以对比一下deque）

3. 你需要频繁的在迭代的同时删除数据，或者你需要频繁的合并容器。`std::list`因为本身数据是离散存储的，所以迭代中删除数据不会导致后续的迭代器的失效，做区间插入的时候也可以保证全局的异常安全性。

## std::list 中特殊的函数

`std::list`中有一些特殊的成员函数值得我们在这里稍微的讨论一下：

### size

这个函数比较特别的是，它的开销可能是O（N），在C++11之前，标准规定它的开销可能是O（N）也可能是O（1），所以轻易不要调用这个函数。比如

```C++
if (list.size() == 0)
```

最好写成

```C++
if (list.empty() == 0)
```

### remove

这个函数之所以特殊是因为`std::vector`不提供这个函数，而是使用算法`std::remove`，而remove实际上不删除数据，需要配合`std::vector::erase`来删除数据。而 `std::list` 的提供这个成员函数而且会实际删除数据。

### insert(iterator pos, InputIt first, InputIt last)

这个函数之所以特殊是因为所有的容器中的区间insert，只有`std::list`的这个方法保证强异常安全性（要么全部插入，要么一个都不插入）【2】


# std::forward_list

前面提到`std::list`是一个双向链表，在C++11之后，STL还提供了单链表：`forward_list`。单链表的开销比双链表要小一些，但是舍弃了双向迭代的功能，而且只支持在头部插入和删除数据。

在你不需要双向迭代的时候，你可以考虑使用单链表替代双链表，比如哈希表的冲突列表就可以使用单链表来实现的。

## insert_after、erase_after

这两个函数和其实容器不太一样，其他容器是在给定的pos之前（实际上给定的位置，但是因为当前位置的数据往后挪动，相当于插入到来这个位置的元素之前）插入删除，单链表因为不支持双向迭代，只能实现在给定的位置之后插入和删除。

---

【1】 C++标准只规定了结果，并不规定如何实现，不同的C++编译器对于`std::vector`的实现可能完全不一样，这里看的源码是来自llvm的`libcxx`实现。

【2】单链表也支持类似功能，不过它提供的方法是 `insert_after`，不是`insert`
