---
title: std::bind 的实现原理
date: 2017-01-27 11:43:06
tags:
 - C/CPP
---

C++11 中有一个非常神奇的函数`std::bind`，它让你可以对函数进行适配，动态的绑定参
数。比如你有一个函数接收两个参数，一个算法接收单参数的`callable object`，那么通
过`std::bind`我们可以让两者协同工作。

```c++
void Foo(int a, int b);

template< class InputIt, class UnaryFunction >
UnaryFunction for_each( InputIt first, InputIt last, UnaryFunction f );

std::vector<int> v;
int b = 10;
for_each(v.begin(), v.end(), std::bind(Foo, _1, b))
```

那么这个神奇的 `bind` 函数和神奇的占位符到底是怎么实现的呢，它们的工作原理是什么
？这个问题困扰我很久，最近在网上找到一篇讲解非常清晰的[文章][ref]，这里对其中的
内容做简单的摘要和翻译，希望对于大家理解背后的工作原理会有帮助。

[ref]: https://accu.org/index.php/journals/1397

<!--more-->

# bind 是一个工厂方法

很显然 bind 是一个工厂方法，因为我们传递给 for_each 的是 bind 的返回值而不是
bind 本身。bind 创建的对象在 boost 实现中叫做 bind_t。因为 bind 要做的事情是适配
器，所以它返回的对象必然和它接受的对象是一样的—— `callable object`。因此，bind_t
中必然重载了函数调用符。

```c++
class bind_t {
public:
    template<typename A>
    operator()(A a);
};
```

此外，bind_t 需要以用户给定的参数调用原本的函数，所以它的内部实际还存储了另外两
个成员，也就是原本的函数和用户已经确定的参数。

```c++
template <typename F, typename L>
class bind_t {
    F f_;
    L l_;
public:
    bind_t(F f, L l) : f_(f), l_(l) {}

    template<typename A>
    operator()(A a);
};
```

# 参数

对于`std::bind`来说，参数分为两种，一种是用户创建`bind_t`的时候提供的，另一种是
调用 bind_t 的`operator()()`的时候提供的，前者在创建 bind_t 的时候就已经知道，而
后者是在调用`bind_t`的`operator()()`的时候才知道，为了方便描述我们把它们分别叫做
L 和 A 。

很显然，L 和 A 都可能有多个，多个 A 可以通过重载不同版本的 operator() 来解决，比
如：

```c++
template <typename F, typename L>
class bind_t {
    F f_;
    L l_;
public:
    template<typename A>
    operator()(A a);            // 单个参数

    template<typename A1, typename A2>
    operator()(A1 a1, A2 a2);   // 两个参数

    ...
};
```

但是多个 L 却不行，因为类是没有办法重载的，你不能既定义 bind_t 有两个模板参数又
定义它有三个模板参数。而且就算你可以这么做，这种方式也是一种不太合理的做法，因为
这样会导致 L 和 A 进行排列组合，实现起来将会极其复杂，解决办法是归一，增加一层
间接性，使用 List 而不是使用单个元素。

```c++
template<typename A1>
class List1 {
    // 省略了构造函数
    A1 a1;
};

template<typename A1, typename A2>
class List2 {
    // 省略了构造函数
    A1 a1;
    A2 a2;
}
```

这样一来我们就可以使用 List1 和 List2 作为 bind_t 的参数从而解决 L 有多个的问题
。所以实际上，bind 这个函数的工作就是做 List 的封装以及对应 bind_t 的创建。

```c++
template <typename F, A1, A2>
bind_t<F, List2> bind(F f, A1 a1, A2 a2) {
    List2 list(a1, a2);
    return bind_t<F, List2>(f, list);
}
```

当然为了能够支持多个参数，实际上 bind 是一个系列的模板函数的重载。

# bind_t 的 operator() 的实现

有了上面的基础之后，我们来看operator()的具体实现，首先我们需要知道的是，在
bind_t中，我们并不知道 L 到底是几个参数（就像你在 vector 的定义中不可能知道你存
储的到底是什么类型）。所以我们没有办法在 bind_t 中去处理参数绑定的问题，相反我们
需要让 L 去处理参数的绑定问题。

```c++


template<typename F, typename L>
class bind_t {
    F f_;
    L l_;
public:
    template<typename A>
    operator()(A a) {
        l_(f, a);
    }
};
```

也就是说，List 也必须提供函数调用操作符。

```c++
template<typename A1, typename A2>
class List2 {
public:
    template<typename F, typename A>
    operator()(F f, A a);

private:
    A1 a1;
    A2 a2;
}
```

我们之前说过，bind_t 为了能够支持多个参数的调用，重载了多个 `operator()` 而这些
重载的函数如果按照前面的实现方式——调用 List 对应的`operator()`的话就会导致 List
也需要重载多个 `operator`，而这无疑是非常繁琐的事情。为了解决这个问题，同样可以
用列表替代单个参数。也就是像下面这样实现 bind_t 的函数调用操作符。

```c++
template<typename F, typename L>
class bind_t {
    F f_;
    L l_;
public:
    template<typename A>
    operator()(A a) {
        List1 list(a);
        l_(f, list);
    }

    template<typename A1, typename A2>
    operator()(A1 a1, A2 a2) {
        List2 list(a1, a2);
        l_(f, list);
    }
};
```

# ListN 的 operator() 的实现

**ListN 表示 List1，List2，List3 中的任何一个，这里一 List2 为例**

List2 需要完成两件事情，完成参数的绑定和调用实际的函数 f。所以 operator() 最终看
起来应该是这个样子。

```c++
template<typename A1, typename A2>
class List2 {
public:
    template<typename F, typename L>
    operator()(F f, L l) {
        f(l1, l2);  // 这里的 l1，l2 是根据 L 推导出来的
    }

private:
    A1 a1;
    A2 a2
}
```

所以我们剩下的问题是如何根据 l，a1，a2 最终推导出 f 的实际调用参数 l1，l2。其实
这个算法很简单。以 A1 为例，如果 A1 是普通的值，那么 l1 == a1。如果 A1 是一个占
位符，那么 l1 就等于 l 中对应的值。boost 在实现这个逻辑的时候使用了一个非常巧妙
的方式——函数重载。它重载了 List 的 `[]` 操作符，然后根据参数的类型来判断返回什么
值。

```c++
template<typename A1, typename A2>
class List2 {
public:
    template<typename F, typename L>
    operator()(F f, L l) {
        f(l[a1], l[a2]);  // 这里的根据 a1，a2 的类型得到实际的值
    }

private:
    A1 a1;
    A2 a2
}

template<typename A1>
class List1 {
public:
    A1 operator[](placeholder<1>) const { return a1; } // 如果是占位符
    template <typename T>
    T operator[](T v) const { return v; }   // 如果是普通的类型

private:
    A1 a1;
};
```

这里的实现使用了一个C++比较偏门的特性，在重载解析的时候，普通函数的优先级高于模
板函数，也就是说当遇到类型为 `placeholder<1>` 的参数时候，虽然模板函数也可以实例
化出正确的函数，但是因为有一个不需要要实例化的普通函数存在，重载解析会选择调用普
通的函数，也就是调用返回占位符对应的值的那个函数。

# placeholder<1> 的作用

从上面的代码我们可以看出，实际上，placeholder<1> 只是用来做重载解析的分派用的，
我们需要的是它的类型而不是它的值，所以你会发现前面 `operator[]` 甚至没有给出参数
名称。placeholder<1> 的定义非常简单：

```c++
template<int I>
class placeholder{};
placeholder<1> _1;
placeholder<2> _2;
placeholder<3> _3;
```

这种把数值当类型的技巧可以参考《C++设计新思维》一书。相信现在你应该很清楚
`std::placeholder::_1` 是什么东西了吧。


# 实际例子

为了方便理解这个参数绑定的过程，我们以文章开头的例子来详细分析一下：

```c++
void Foo(int a, int b);

for_each(v.begin(), v.end(), std::bind(Foo, _1, b));
```

这个例子中，std::bind 返回的 bind_t 类型是

```c++
bind_t<void(int, int), List2<placeholder<1>, int>> binder = {
    Foo,          // f_
    {_1, 10}      // l_
}
```

现在我们用单参数调用 `binder`：

```c++
binder(5);
```

那么实际上调用的代码是 binder 的 `operator()(int a)` 函数：

```c++
class bind_t<void(int, int), List2<placeholder<1>, int>> {
public:
    operator()(int a) {
        List1<int> list(5);
        l_(f_, list)
    }
};
```

然后调用了 `List2<std::placeholder, int>` 的 `operator()(List1 list)` 函数：

```c++
class List2<std::placeholder<1>, int> { // binder 中的成员变量 l_
public:
    operator()(void(*f)(int, int), List1 list) {
        f(list[a1], list[a2])
    }
private:
    placeholder<1> a1;      // 占位符 _1
    int a2                  // 创建 binder 的时候提供的参数 10
};
```

最终调用了List1<int>的`operator[](placeholder<1>)`函数和`operator[](int a)` 函数
：

```c++
class List1<int> {  // bind_t 的 operator()(int) 中创建的 local 变量
    operator[](placeholder<1>) {
        return a1;      // 返回 5
    }

    operator[](int a) {
        return a;       // 返回 10
    }

private:
    int a1;     // 这个是函数调用的实参也就是 binder(5) 调用中的 `5`
};
```

# 结束语

实际上`std::bind`的实现方式和这里提到的有些许出入，比如说为了提高效率，很多地方
都是使用引用而不是值，但是实现原理上应该八九不离十了，boost的实现放在
`boost/bind/bind.hpp` 中，读者可以参考一下。
