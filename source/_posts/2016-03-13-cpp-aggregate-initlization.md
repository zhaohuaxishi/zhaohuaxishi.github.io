title: C++中的聚合初始化
date: 2016-03-13 15:21:58
tags:
 - C/CPP
---

C++标准库中提供两个容器表示数组这个概念：array表示静态数组，vector表示动态数组
。这两个类型都可以使用列表初始化（list initilization）来初始化。

```cpp
std::array<int, 5> = {1, 2, 3, 4, 5};
std::vector<int>   = {1, 2, 3, 4, 5};
```

乍看上去没什么好奇怪的，毕竟都是 STL 中的容器，提供类似的初始化接口没有什么特
别的，而且 C++11 中提供的 `initilizer_list` 也让上面这种语法变得非常的普遍，接
受一个 `initilizer_list` 参数的构造函数甚至有一个专有术语 `initializer-list
constructor`，比如`vector`就有这样一个构造函数：

```cpp
vector( std::initializer_list<T> init,
        const Allocator& alloc = Allocator() );
```

这一切看上去都合情合理，直到我发现，`array` 没有显式定义任何的构造函数。也就是
说它的这种初始化根本就没有用到 C++11 中的任何特性，你甚至可以在 C++98 中使用这
样的语法初始化它。

那么这一切又是如何做到的呢？答案是它使用了 C++ 的另一个特性：聚合初始化（
`aggregate initialization`）。

<!--more-->

# 聚合初始化

聚合初始化其实由来已久，在`C`语言中就存在了。

```cpp
int array[5] = {1, 2, 3, 4, 5};
```

在`C++`中对于聚合体（aggregate）的初始化称为聚合初始化，可以使用上面这种语法。
有两种类型的对象被称为聚合体：

1. 数组类型
2. 满足下列条件的类类型（通常是结构体（struct）或者联合体（union））：
    - 没有私有或保护的非静态数据成员
    - 没有用户提供的构造函数
    - 没有基类
    - 没有虚函数

所以说下面这个结构体的对象可以使用聚合初始化：

```cpp
struct Aggregate {
    int i;
    int j;
};

Aggregate aggr = {1, 2};
```

上面这些都没什么神奇的，真正神奇的是如果你的聚合体中间有嵌套，你可以不用使用花
括号分割：

```cpp
struct Aggregate {
    int arr[4];
    int j;
};

Aggregate aggr = {1, 2, 3, 4, 5};
```

在上面这个初始化中，`arr` 成员会得到`{1, 2, 3, 4}`, 而 `j` 成员会初始化成 `5`
。

# std::array 是一个聚合体

所以其实 `std::array` 之所以可以使用列表初始化的原因是它是一个聚合体，也就是说
这个模板的所有成员都是 `public`，理论上你可以直接访问他们，不过C++标准没有规定
它的成员变量的名称，使用他们是未定义的行为，不具有可移植性。
