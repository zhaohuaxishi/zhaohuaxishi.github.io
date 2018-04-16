---
title: C++轮子——函数式编程
date: 2018-04-16 09:30:44
tags:
 - C/CPP
---

# 仿函数

算法库中有很多算法都有一个重载的版本，接收一个判断条件，用于区间内部对元素进行判断，这些版本通常都以`_if`结尾，比如：`std::find_if`，`std::copy_if`（也有一些不是，比如`accumulate`）。

```
template< class InputIt, class T >
InputIt find( InputIt first, InputIt last, const T& value );

template< class InputIt, class UnaryPredicate >
InputIt find_if( InputIt first, InputIt last,
                 UnaryPredicate p );
```

这个判断条件，语法上需要是一个`callable object`，它可以是普通的函数，也可以是一个重载了函数调用符的类型对象。这个可调用对象，如果返回值是`bool`则称之为`predicate`，很多人翻译成谓词。

我们把重载了函数调用符的对象称为仿函数，它在功能上是一个函数，在语法上是一个类。它和普通函数最大的区别是它可以保存内部状态。

加入我们现在要找出第一个奇数，我们可以这样写。

```
bool is_odd(int a) { ... }
std::find_if(std::begin(a), std::end(a), is_odd)
```

但是加入我们

```
bool is_even(int a) { ... }
std::find_if(std::begin(a), std::end(a), is_even)
```

但是如果我实现成仿函数，我们可以这样写：

```
class NumberPredicate {
public:
    NumberPredicate(bool found_odd) { ... }

    bool operator()(int a) const {
        if (found_odd_) {
            ...
        }

        ...
    }

private:
    bool found_odd_;
};

std::find_if(std::begin(a), std::end(a), NumberPredicate());
```

上面这个例子其实并不太恰当，我不推荐在一个函数中实现两个功能，但是它展示了仿函数区别于普通函数的重要特性——可以保存状态。

# 未完待续

## std::function, std::bind, lambda

