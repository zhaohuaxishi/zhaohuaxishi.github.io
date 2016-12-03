title: 读书札记之 —— 《More Exceptional C++》
tags:
  - C/CPP
  - 读书笔记
date: 2016-04-22 17:14:09
---

和《More Effective C++》中的`more`不一样的是，这本书的`more`体现在广度上而不是
深度上。《Exceptional C++》一书的已经是非常有深度的一本书了（或者说是比较难的一
本书了），这本书是前者在其他内容上的扩展。

这本书涉及到很多内容，包括范型模板、性能调优等等方面。个认识收获最多的是
`traits`类的使用。

<!--more-->

# `traits` 类

在作者的观点中，获得多态行为的方式有四种，虚函数、模板、重载、转换。但是大部分
人提到多态的时候只会想到第一种方式。转换是一种不太建议使用的机制，没有太多讨论
的必要，而重载更多的人喜欢把它当成一个单独的特性而不是多态的一种形式。

剩下两种其实都可以算是大家都普遍接受的观点，虚函数提供运行期多态而模板提供编译
期多态。单从效率上来说，编译器多态也可以算是一种非常值得学习的技术。依我个人观
点来看，虚函数的关键在于`override`，模板的关键则在于`特化`。这也是`traits`类强
大之所在。比如为了多态的实现一个`Create`，虚函数的写法可能是这样的：

```cpp
class WidgetCreator {
public:
    virtual Widget* Create() {
        return new Widget();
    }
};

class PrototypeWidgetCreator : public WidgetCreator { // 继承
public:
    // 其他代码初始化 prototype_
    virtual Widget* Create() {
        return prototype_->Clone();
    }

private:
    Widget *prototype_;
};
WidgetCreator createor;
Widget* widget = createor.Create();

PrototypeWidgetCreator prototype;
Widget* special_widget = prototype.Create();
```

使用`traits`类是写法可能是下面这样的：

```cpp
template <typename T>
class WidgetCreator {
public:
    static T* Create() {
        return new T();
    }
};

template<>
class WidgetCreator<SpecialWidget> { // 特化
public:
    // 其他代码初始化 prototype_
    static Widget* Create() {
        return prototype_->Clone();
    }

private:
    Widget *prototype_;
};

Widget* widget = WidgetCreator<Widget>::Create();
Widget* special_widget = WidgetCreator<SpecialWidget>::Create();
```

这两种写法在效果上并不会有太多的区别，真正能够区分两者的地方在于扩展性。虚函数
需要使用继承而`traits`并不需要，要知道继承是一种非常强的耦合关系，所以从代码的
耦合度上来说，`traits`要略胜一筹，而且从效率上来说因为`traits`使用的是编译时多
态，所以理论上（我没有做个实现，所以说理论上）速度应该会更快。

# 构造函数的异常安全

《Exceptional C++》一书中讨论了很多异常安全的内容，主要涉及到拷贝构造函数和赋值
操作符。本书中添加了额外的异常安全内容，包括普通的构造函数、参数求值、`pimpl`、
继承等对于异常安全的影响。

## 构造函数

第一个需要记住的是，永远不要在初始化列表中获取未管理的资源，比如：

```cpp
class Widget {
public:
    Widget() : t_(new T()), z_(new Z()) { }
private:
    T* t_;
    Z* z_;
}
```

这种方式不是异常安全的，即使我们使用 `function try block`，我们依旧没有办法在
`z_` 创建抛出异常的时候释放掉已经分配的 `t_`。正确的方式是：

```cpp
class Widget {
 public:
    Widget() {
        t_ = new T();
        try {
            z_ = new Z();
        } catch (...) {
            delete t_;
            cout << "widget error" << endl;
            throw;
        }
    }

 private:
    T* t_;
    Z* z_;
};
```

或者更好的使用智能指针取代原来的裸指针。

## 参数求值顺序

永远不要写：

```cpp
f (new T1, new T2)
```

这样的代码，你应该写成：

```cpp
uniuqe_ptr<T1> t1(new T1);
uniuqe_ptr<T2> t2(new T2);
f(move(t1), move(t2));
```

## Pimpl 和异常安全

`Pimpl` 的强大在 《Exceptional C++》一书中有详细的介绍，本书中介绍了`Pimpl`的另
一个功能，那就是异常安全。当然这不是`Pimpl`的功能，是指针的功能。普通的值没有办
法达到异常安全所需要的不抛异常的`Swap`，但是指针可以，所以`Pimpl`可以达到很好的
异常安全性。

## 继承和异常安全

继承是一种很强的关系，实际上的派生类中包含基类的子对象，伪代码如下：

```cpp
class Drived {
private:
    Base base_;
};
```

这样一来，只要`base_`无法异常安全的拷贝的话，`Drived`的对象就无法异常安全的拷贝
。如果不是只用继承，改用组合的话，你可以得到类似下面的 伪代码。

```cpp
class Drived {
private:
    Base* base_;
};
```

这样一来，由于指针的异常安全性，你可以轻松的实现异常安全的`Drived`拷贝。这里指
出了用组合强于`private`的另一方面。

# 优化

优化的第一原则永远都是：

> 不要做优化

内联是一种优化，所以除非你有十足的证据显示内联可以很大的提升你的程序的性能，否
则，你不需要把函数声明成内联。

`cow`是一种优化，但是在多线程的环境下，这种优化是得不偿失的，这也是为什么在
`C++11`标准中不允许使用引用计数来实现`std::string`的原因。

# 分解 MI 带来的连体婴儿

本书中提供了一种非常优化的方式解决下面的问题：

```cpp
class BaseA {
public:
    virtual void foo(); // A 功能
}

class BaseB {
public:
    virtual void foo(); // B 功能
}

class Drived : public BaseA, public BaseB {
public:
    virtual void foo(); // 一个函数完成两个功能？？？？
}
```

# ValuePtr

关于智能指针，这本书对于`ValuePtr`的讨论是非常值得学习，其中使用到的`traits`技
术和模板构造函数技术，也让这个设计变得非常的有吸引了。
