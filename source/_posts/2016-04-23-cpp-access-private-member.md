title: C++中访问私有成员的方法
date: 2016-04-23 15:55:29
tags:
 - C/CPP
---

今天读《Exceptional C++ Style》一书的时候，发现在C++中竟然存在一种合法的方式去
访问对象的私有成员，想起以前和嘉伟讨论过了调用私有函数的问题，猛然发现方法不止
原来一种，这里做一个总结。

这三种方法都是使用一个间接层，现在想来果然还是那句老话，没有什么问题是通过间接
层无法解决的。

当然，这里说的所有内容，基本上纯属个人找乐子而已，你在实际编码中不到万不得已千
万不要去使用这些方法，它们给程序的安全和耦合性上都有非常大的危害。

<!--more-->

# 方法一 —— 使用模板函数的特化

通过模板成员函数的特化，我们可以访问类的私有成员，这基本上可以说是语言的一个漏
洞。

```cpp
#include <iostream>

using namespace std;

class Widget {
 public:
    template <typename T = void>
    void AccessPrivateFunc() {
        cout << "Default —— Do Nothing" << endl;
    }

 private:
    void PrivateFunc() { cout << "Call Private Function" << endl; }
};

struct PrivateAccessor {};

template <>
void Widget::AccessPrivateFunc<PrivateAccessor>() {
    PrivateFunc();
}

int main(void) {
    Widget widget;
    widget.AccessPrivateFunc();
    widget.AccessPrivateFunc<PrivateAccessor>();
    return 0;
}
```

这段程序是输出如下：

```cpp
Default —— Do Nothing
Call Private Function
```

为了确保`PrivateAccessor`的唯一性，你可以把它放入到匿名命名空间中。此外要注意的
就是这段代码使用了函数模板的默认参数，这个特性只存在于C++11中。当然你可以通过控
制`PrivateAccessor`的可见性来控制哪些函数可以访问`private`成员。

# 方法二 —— 通过额外的友元类

这种方式通过增加一个间接的基类`AccessController`来完成私有函数的访问，通过控制
这个基类你可以控制哪些私有成员可以访问哪些不行。

```cpp
class Widget {
 private:
    void PrivateFunc() { cout << "Call Private Function" << endl; }
    friend class AccessController;
};

class AccessController {
 protected:
    void AccessPrivateFunc(Widget& w) { return w.PrivateFunc(); }
};

class Client : private AccessController {
 public:
    using AccessController::AccessPrivateFunc;
};

int main(void) {
    Client client;
    Widget widget;
    client.AccessPrivateFunc(widget);
    return 0;
}
```

上面这段代码的输出是：

```cpp
Call Private Function
```

# 方法三 —— 使用成员函数指针

`private` 限制了名字的可见性，使用函数指针规避名字可见性，我们一样可以访问私有
方法。

```cpp
class Widget;
typedef void (Widget::*PMember)();

class Widget {
public:
    PMember AccessPrivateFunc() {
        return &Widget::PrivateFunc();
    }

private:
    void PrivateFunc();
};

int main() {
   Widget widget;
   PMember p = widget.AccessPrivateFunc();
   (widget.*p)(); // call private function
}
```
