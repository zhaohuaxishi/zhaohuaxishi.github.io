---
title: 捉妖记之——私有继承
date: 2018-12-09 11:51:25
tags:
  - 捉妖记
---

C++的集成比大部分的语言不同的地方在于它的集成也有访问权限。通常来说用得比较广泛的两种是，公有继承和私有继承。虽然他们都是继承，但是实际上有非常本质的区别。《Effctive C++》中有提到，私有继承实际上是`has-a`的关系，而公有继承才是真正传统意义上的继承，也就是`is-a`的关系。

<!--more-->

我们常说C++的动多态使用继承来实现，但是实际上更准确一点的说法应该是，C++中的动多态使用公有继承来实现，因为私有继承本质上来说是组合关系而不是继承关系，我们没有办法用父类的引用指向一个私有继承的子类对象。也就是说下面这段代码实际上无法通过编译。

```cpp
class Base {
};

class Drived : Base {
};

int main(int argc, char* argv[]) {
    Drived drived;
    Base& base = drived;    // 编译错误
}
```

这个错误其实并不难，因为编译时期就会报错，不会有太大的问题。我最近碰到编译可以正常通过，但是运行的时候会抛异常的BUG，花了一些时间定位，记录在这。

# std::enable_shared_from_this

在C++11中，我们常常使用智能指针来避免资源泄露或者野指针的问题，其中基于引用计数实现的`std::shared_ptr`有一个非常有趣的功能是可以实现获取`this`的智能指针。

我们知道，每个成员函数，都默认会有一个`this`参数指向当前对象的地址。但是这个指针是一个裸指针，使用不当会有野指针的风险，比如下面这段代码。

```cpp
std::vector<std::function<void(void)>> actions;

class Widget {
public:
    void RegisterAction() {
        actions.push_back(std::bind(&Widget::Action, this));
    }

    void Action() {
        // do something
    }
};

void RegisterAction() {
    Widget widget;
    widget.RegisterAction()
}

int main(int argc, char* argv[]) {
    RegisterAction();

    for (const auto& action : actions) {
        action();
    }
}
```

在`Widget::RegisterAction`中，我们把this指针传递给了binder，生成了一个`std::function`放在全局的`actions`中（我并不推荐使用全局变量，这里只是为了方便演示而已，实际代码中尽量不用全局便利），这个`std::function`对象的生命周期比`widget`本身要长，所以在`main`函数中，会产生野指针崩溃。

为了避免野指针，我们第一个想到的就是智能指针，搭配`std::enable_shared_from_this`，它可以获取到`this`的指针指针。所以上面这一段代码可以这样更改。

```cpp
class Widget : std::enable_shared_from_this<Widget> {                      // 1
public:
    void RegisterAction() {
        actions.push_back(std::bind(&Widget::Action, shared_from_this())); // 2
    }

    void Action() {
        // do something
    }
};

void RegisterAction() {
    auto widget = std::make_shared<Widget>();                              // 3
    widget->RegisterAction()
}
```

注意上面的注释处的修改，首先如注释`// 1`所示，我们要继承自`std::enable_shared_from_this`，然后如注释`// 2`所示，使用`shared_from_this`获取到指向自身的智能指针。需要特别注意的是，我们必须保证`Widget`的对象是通过只能指针构造出来的，如注释`//3`标注所示。

# 但是

上面这段代码可以通过编译，但是在运行的时候会抛`bad_weak_ptr`异常。上面这段代码逻辑上没有什么问题，真正的错误在于继承上，我们使用`std::enable_shared_from_this`的时候，必须公有继承自它，而`class`的默认的继承是私有继承。下面这段描述来自`cppreference`

> Publicly inheriting from std::enable_shared_from_this<T> provides the type T
with a member function shared_from_this

注意最前面的`Public inheriting`。

# 为什么

实际上，`enable_shared_from_this`类的内部，保存了一个weakptr，这个weakptr会在构造`shared_ptr`的时候被设置，下面是来自`boost`中的源码：

```cpp
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

如果我们使用的是公有继承，我们可以使用多态，把自己绑定到第一个模板函数的最后一个参数`pe`上面，进而进一步的设置`pe`内部的`weakptr`，否则我们会`fallback`到后面那个函数，导致`enable_shared_from_this`内部的`weakptr`未初始化成正确的至，最终我们调用`shared_from_this`的时候会尝试从这个未初始化成正确的值的`weakptr`构造一个`shared_ptr`，进而导致`std::bad_weak_ptr`的异常的产生。
