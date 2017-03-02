---
title: enable_shared_from_this 的使用及实现原理
date: 2017-01-25 22:36:02
tags:
 - C/CPP
---

**本文除 boost 源码以外，大部分源码都是伪代码，为了能够更清晰的理解问题，去掉了
模板相关的内容。文中很多地方直接使用了 sp 一词表示 shared_ptr**

使用智能指针难以避免的场景之一就是需要把 this 当成一个 shared_ptr 传递到其他函数
中去。比如你的类实现了某个接口，而你需要把自己注册为该接口的实现。你不能简单粗暴
的用 this 指针构造一个 shared_ptr，因为那样做会导致两个独立的 shared_ptr 指向同
一份资源，这对于基于引用计数的 shared_ptr 来说是致命的。比如下面的代码。

```c++
class Widget {
    std::shared_ptr<Widget> GetPtr() {
        return shared_ptr<Widget>(this);     // 错误
    }
};

int main() {
    auto widget = std::make_shared<Widget>();
    widget->GetPtr();
}
```

正确的打开方式是继承来自 std::enable_shared_from_this，然后调用它的
shared_from_this 成员函数

```c++
class Widget : public std::enable_shared_from_this<Widget> {
    std::shared_ptr<Widget> GetPtr() {
        return shared_from_this();
    }
};

int main() {
    auto widget = std::make_shared<Widget>();
    btn->GetPtr();
}

```

把自己作为基类的模板参数看起来非常诡异，它有一个更诡异的名字——奇异递归模板模式。
关于它的使用这里不过多的介绍，有兴趣的同学可以参考《More Effective C++》，《C++
Templates》等书籍。这里主要探讨的是它是如何实现返回自身的智能指针的。

<!--more-->

# 避免多个独立的 shared_ptr 指向同一份资源

在之前的错误代码中，有两个独立的 shared_ptr 指向同一块空间

```c++
shared_ptr<Widget>(this);   // 成员函数中
std::make_shared<Widget>(); // main 函数中
```

很显然 `main` 函数中这这个 shared_ptr 必不可少，所以我们只能去掉
`shared_ptr<Widget>(this)` 而让它共享 main 函数中创建的这个 shared_ptr。

# 如何共享 make_shared 的返回的资源

实际上，抛开一切来讲，我们想要的不是基于 this 去创建一个 shared_ptr<Widget>，而
是希望 this 的类型就是 shared_ptr<Widget>。想通了这一点你就会发现，其实我们应该
从 shared_ptr 类入手，而不是 Widget 类，因为只有 shared_ptr 类中 this 的类型才是
shared_ptr。

make_shared 会调用 shared_ptr 的构造函数，在 shared_ptr 的构造函数中，this 的类
型就是 shared_ptr<Widget>，所以从逻辑上分析，我们甚至可以这样去实现
shared_from_this 函数

```c++
// 下面所有的代码省去模板相关的代码，以便阅读。

class shared_ptr<Widget> {
public:
    explicit shared_ptr<Widget>(Widget* w) {
        // 其他初始化，可能在初始化列表中而不是构造函数内部
        RegisterSharedPtr(w, this);
    }
};

void RegisterSharedPtr(Widget* w, shared_ptr<Widget>* sp) {
    // 建立 w 到 sp 的映射关系
}

void GetSharedPtr(Widget* w) {
    // 查找 w 对应的 sp
}

class Widget {
public:
    shared_ptr<Widget> shared_from_this() {
        GetSharedPtr(this);
    }
};
```

# 建立 w 和 sp 之间的映射关系

建立 w 和 sp 的关系最直接的方式是使用一个 map 来集中管理，另一种方式就是采用分布
式的管理方式，也就是让 w 直接存储这个 sp，而不用借助额外的 map。

```c++
class shared_ptr<Widget> {
public:
    explicit shared_ptr<Widget>(Widget* w) {
        // 其他初始化
        if (w) {
            w->SetSharedPtr(this);
        }
    }
};

class Widget {
public:
    void SetSharedPtr(shared_ptr<Widget>* sp) {
        sp_ = *sp;
    }

    shared_ptr<Widget> GetPtr() {
        return sp_;
    }

private:
    shared_ptr<Widget> sp_;
};
```

# 抽象公共基类

上面的设计的最大的问题是，你需要在自己的内部定义一个 shared_ptr 类型的成员变量以
及提供一个 GetPtr 和 SetSharedPtr 成员方法，这些代码和 Widget 类的业务逻辑可能没
有太大的关系，而且不适合重用。为了抽象并重用这些代码，我们可以抽象出一个公共的基
类，这也就是 std::enable_shared_from_this 实际上做的事情

```c++
class enable_shared_from_this {
public:
    void SetSharedPtr(shared_ptr<Widget>* sp) {
        sp_ = *sp;
    }

    shared_ptr<Widget> GetPtr() {
        return sp_;
    }

private:
    shared_ptr<Widget> sp_;
};

class Widget : public enable_shared_from_this {
};

class shared_ptr<Widget> {
public:
    explicit shared_ptr<Widget>(Widget* w) {
         w->SetSharedPtr(this);
    }
};
```

上面这段代码最大的漏洞在于，shared_ptr 是一个模板，它并不知道 Widget 类是否继承
自 enable_shared_from_this，所以 `w->SetSharedPtr(this)` 这一句的调用不完全正确
。boost 中解决这个问题使用了一个非常巧妙的方法——重载，通过重载我们可以让编译器自
动选择是否调用 SetSharedPtr。

```c++
class shared_ptr<Widget> {
public:
    explicit shared_ptr<Widget>(Widget* w) {
        SetSharedPtr(this, w);
    }
};

void SetSharedPtr(shared_ptr<Widget>* sp, enable_shared_from_this* w) {
    w->SetSharedPtr(sp);
}

void SetSharedPtr(...) {
    // 什么也不做
}
```

这段代码的精妙，让人叹为观止。

# 最后一个问题

上面这些代码的逻辑上是正确的，但是实际上还有一个巨大的BUG，那就是 Widget 的内部
存在一个指向它自己的 shared_ptr<Widget>，这意味着你永远无法销毁掉 Widget。销毁
Widget 的前提是没有 shared_ptr 指向它了，而销毁 Widget 必然需要销毁它的成员变量
，包括指向自身的那个  shared_ptr，而它的存在又否定了我们能销毁 Widget 的前提——没
有 shared_ptr 指向它。这就像是你在画了一个指向自身的箭头，它让你自身形成了循环依
赖，永远没有办法跳出来。

普通的 shared_ptr 循环依赖的问题的处理通常使用的是 weak_ptr，这一次也不例外。我
们需要存储一个 weak_ptr 作为成员变量，而不是 shared_ptr，然后在需要的时候通过
weak_ptr 构建出 shared_ptr，所以正确的打开方式是：

```c++
class enable_shared_from_this {
public:
    SetSharedPtr(shared_ptr* sp) {
        wp = sp
    }

    shared_ptr shared_from_this() {
        return shared_ptr(wp);
    }

private:
    weak_ptr wp;
}
```

这是一个接近正确答案的写法，也是一个比较容易理解的方法，但和实际上的写法还一些细
微的差别，比如实际上 SetSharedPtr 中并没有把 sp 直接赋值给 wp，而是使用了
shared_ptr 的`别名构造函数`，为什么这么写，个人能力有限，还没弄清楚，弄懂了再回
来补充。

# boost 的源码片段

上面的所有代码都是个人按照自己的理解写出来的，为了方便理解，这里给出 boost 的相
关源码片段，以供对比参考（需要注意的是 pn 和引用技术有关，但是和这里讨论的
enable_shared_from_this 没有太大的关联，你可以忽略它）：

```c++
template<class T> class shared_ptr
{
    template<class Y>
    explicit shared_ptr( Y * p ): px( p ), pn() // Y must be complete
    {
        boost::detail::sp_pointer_construct( this, p, pn );
    }
};
```

```c++
template< class T, class Y > inline void sp_pointer_construct( boost::shared_ptr< T > * ppx, Y * p, boost::detail::shared_count & pn )
{
    boost::detail::shared_count( p ).swap( pn );
    boost::detail::sp_enable_shared_from_this( ppx, p, p );
}
```

```c++
template< class X, class Y, class T > inline void sp_enable_shared_from_this( boost::shared_ptr<X> const * ppx, Y const * py, boost::enable_shared_from_this< T > const * pe )
{
    if( pe != 0 )
    {
        pe->_internal_accept_owner( ppx, const_cast< Y* >( py ) );
    }
}

inline void sp_enable_shared_from_this( ... )
{
}
```

```c++
template<class T> class enable_shared_from_this
{
public:

    shared_ptr<T> shared_from_this()
    {
        shared_ptr<T> p( weak_this_ );
        BOOST_ASSERT( p.get() == this );
        return p;
    }
public: // actually private, but avoids compiler template friendship issues

    // Note: invoked automatically by shared_ptr; do not call
    template<class X, class Y> void _internal_accept_owner( shared_ptr<X> const * ppx, Y * py ) const
    {
        if( weak_this_.expired() )
        {
            weak_this_ = shared_ptr<T>( *ppx, py );
        }
    }

private:

    mutable weak_ptr<T> weak_this_;
};
```
