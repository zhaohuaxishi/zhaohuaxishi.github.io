title: C++ 多态性继承和非多态性继承的区别
date: 2016-02-24 09:05:27
tags:
 - C/CPP
---

在 C++ 中多态的实现是以继承为基础的，但是这门语言的演化中继承早于多态出现，继承
不一定就意味着多态的使用[^0]。在后文中我把用于多态的继承称为多态性继承，不用于
多态的继承称为非多态性继承[^1]。

多态被称为面向对象编程的核心，可以比较确定的说在 C++ 中多态性的继承应该比非多态
性的常用的多。那么这两种继承模型之间主要区别是什么呢？这就是本文将要讨论的问题
。

**注意：本文中讨论的关于C++ 多态性和非多态性继承区别并不是C++的一项准则而是一种
最佳实践。**

<!--more-->

# 区别

个人认为多态性继承和非多态性继承的关键区别点在于：两种模型中基类和派生类的作用
不同。

- 在多态性继承中，基类的主要作用是提供接口[^2]，子类的主要作用是提供接口的实现
- 在非多态性继承中，基类的主要作用是提供实现，子类的主要作用是为实现封装接口

# 多态性继承：父类接口、子类实现

这一点应该比较好理解，因为这正是多态的基本功能所在：

```cpp
class Shape {
public:
    virtual double Area() = 0;  // 父类提供接口
};

class Rectangle : public Shape {
public:
    Rectangle(double x, double y) : _x(x), _y(y) { }

    virtual double Area() {
        return _x * _y;   // 子类实现
    }

private:
    double _x;  // 这里只是举例子，矩形的表示通常
    double _y;  // 是通过两个顶点而不是两条边。
};

// 用户代码

Shape* shape = new Rectangle(2, 3);
shape->Area();
```

在多态中我们通常使用的是父类提供的接口多态的调用子类提供的实现，而在非多态性继
承模型中则通常是相反的。

# 非多态性继承：父类实现、子类接口

这一点听起来比较诡异，而且你完全可以写出不符合这种模式的代码，但是从许多最佳实
践的角度来看，确实是如此的，比如使用 `Deque` 来实现一个 `Stack` 和 `Queue`：

```cpp
class Deque {
public:
    void push_back(int val);
    void push_front(int val);
    int pop_back();
    int pop_front();
};

class Stack : private Deque {
public:
    void push(int val) {
        // 这里可以直接写成 push_back 不过为了强调
        // 父类实现特意加上了父类名。

        Deque::push_back(val); // 使用基类的实现
    }

    int pop() {
        Deque::pop_back(); // 使用基类的实现
    }
};

class Queue : private Deque {
public:
    void push(int val) {
        Deque::push_back(); // 使用基类的实现
    }

    int pop() {
        Deque::pop_front(); // 使用基类的实现
    }
};
```

## 非多态性的继承的优势

不使用多态最大的优势在于效率和开销，你不用为每一个对象付出额外的 `vptr` 的空间
开销，也不用为方法的调用付出间接调用的开销。而且你可以让你的类在栈中分配，进一
步提高效率。


## 比非多态性继承更好的方式

注意上面我使用了 private 继承而不是 public 继承，因为我们需要的是父类的实现而不
是它的接口。但是这种方式目前来说并不是最佳的方式，更好的方式是使用组合替代继承
：

```cpp
class Stack {
public:
    int pop() {
        return _container.pop_back();
    }
private:
    Deque _container;
};
```

这种方式比继承有更大的灵活性，因为继承使得 `Deque` 和 `Stack` 这两个类耦合在一
起，而使用组合就没有这个问题。

# 两者其实可以综合

上面描述的只是一种简化现象，在 C++ 的现实世界中事情远远要比这里讨论的复杂的多。

```cpp
class Base {
public:
    virtual bool Init() = 0;
};

bool Base::Init() {
    // 实现父类的初始化
}

class Drived : public Base {
public:
    virtual bool Init() { // 实现父类的接口
        if (!Base::Init()) { return false; }  // 使用父类的实现

        // 初始化派生类
    }
}
```

上面的 `Base` 类中的`Init`[^3]既提供了实现又提供了接口。

[^0]: 很多语言继承意味着多态，因为所有的方法都默认可重载，比如Java和Python。
[^1]: 术语使用的可能并不准确，文中使用这两个术语只是为了方便讨论。
[^2]: 这里说的**主要**是相对于非多态性继承来说的。
[^3]: 你也许会问为什么不用构造函数而自己编写`Init`，这么做的主要原因是很多规范（比如谷歌的C++编码规范）中禁用异常，而构造函数如果出错只能抛出异常，把构造函数的功能弱化，只用于初始化成员变量，而把复杂的初始化功能移动到Init函数中有助于减少构造函数中抛出异常的情况。
