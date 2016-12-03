title: 关于 C++ 11 初始化列表和转换构造函数
date: 2016-03-09 09:52:56
tags:
 - C/CPP
---

C++11 统一了原本各种乱七八糟的初始化规则，采用统一的大括号初始化，所以你可以像
下面这样初始化：

```cpp
string  s;
string  s = "hello world";
string* s = new string("hello world");

string  s{};
string  s = {"hello world"};
string* s = new string{"hello world"};
```

后面的三个大括号初始化都叫做列表初始化。C++ 初始化非常的复杂，详细信息可以参考
下面[这个链接][listinit]。

 [listinit]: http://en.cppreference.com/w/cpp/language/initialization

本文主要讨论 `string s = {"hello world"}` 这种类型的列表初始化和转换构造函数的
关系。

<!--more-->

# 类型转换操作符

在 C++ 中和隐式转换相关的函数有两种，一种是把自身类型转换成其他类型的转换操作符
，比如为了能够像`while(cin >> s)`这样处理输入，<del>标准库中的流可以隐式转换成
`void*` 类型（不默认转换成 bool 的原因是下面这样的代码会变成合法的
`std::cin << 12`）</del>，**前面这种观点是在C++11之前的做法，在C++11中引入了显
式类型转换操作符（explicit convert operator），它使得`std::cin << 12`这种语句变
为非法，因为没有隐式转换，同时又让`while(cin >> s)`这种语句变为合法，因为语言规
定对于到`bool`的转换，在条件语句中（包括，if, for, while 等）会自动完成，所以目
前 stream 的类型转换符声明为`explict operator bool()`**:

```cpp
class basic_ios {
public:
    operator void*() const;			// c++98
    explicit operator bool() const;		// c++11
};
```

# 转换构造函数

另一种是把其他类型转换成自身类型的构造函数，比如下面的简化的 string 定义[^1]：

```cpp
class string {
public:
    string(const char* s);
}
```

在 `C++98` 中这种只需要一个参数就可以调用，而且没有声明成 `explicit` 的构造函
数，我们称之为为`转换构造函数`，这种构造函数使得

```cpp
std::string str = "hello world";
```

这样的语句成为合法的语句，理论上编译器会通过转换构造函数生成一个临时变量，然后
通过拷贝构造函数初始化`str`（不过实际上这个临时变量在绝大部分的编译器中都会优
化掉）。

`C++11` 为了统一初始化的语法，扩展了转换构造函数的定义，只要没有声明为
`explicit` 的构造函数都是转换构造函数。所以下面的代码在 `C++11` 中是合法的：

```cpp
std::pair<int, int> ipair = {1, 2};
```

这是因为 pair 类似下面这样的构造函数：

```cpp
pair(const T1&, const T2&);
```

当编译器看到 `{}` 类型的列表初始化的时候，会查找相应的转换构造函数，然后构造一
个临时变量，然后通过临时变量初始化 `ipair`（当然这也是理论上的，编译器绝大多数
都会把这个临时变量优化掉）。注意这种初始化不允许值的`narrow convert`比如把
`double` 变成`int`, 所以下面的代码会报错。

```cpp
class Narrow {
public:
    Narrow(int a, int b);
};

Narrow narrow = {1, 2.0};
```

# `initilizer_list`

在 C++11 中为了统一初始化语法，还引入了 `initilizer_list` 类型，代表一个同构的
值序列，所以你可以像下面这样写代码：

```cpp
foo(std::initilizer_list<int> list);

foo({1, 2, 3})
foo({4})
```

编译器会自动把大括号内的值封装成一个`initilizer_list`。当然我们也可以让构造函
数接受 `initilizer_list` 作为它的参数。

```cpp
class IntVector {
public:
    IntVector(initilizer_list<int> list);
}
```

这时候我们也可以像下面这样初始化 IntVector：

```cpp
IntVector iv = {1, 2};
```

这种构造函数叫做 `initializer-list constructor`（这个名称我从 [C++11FAQ][faq]
中得知，应该是通用的术语）这种构造函数的优先级要高于普通的构造函数。所以如果我
们还有另外一个构造函数：

 [faq]: http://www.stroustrup.com/C++11FAQ.html#init-list

```cpp
class IntVector {
public:
    IntVector(int a, int b);
    IntVector(initilizer_list<int> list);
}
```

`IntVector iv = {1, 2}` 这条语句调用的是`initializer-list constructor`。


[^1]: 实际上在标准库中`string`不是一个类，而是一个`typedef`。
