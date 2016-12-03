title: C++中构造函数和虚拟函数的微妙关系
date: 2016-01-18 16:33:19
tags:
 - C/CPP
---

构造函数和虚拟函数之间存在许多比较微妙的关系，比如构造函数不能是虚拟函数，构造函
数不能正常调用虚拟函数等等，本文将会讨论产生这些问题的原因。

<!--more-->

# 构造函数为什么不能是虚拟函数

构造函数和析构函数在设计之初就是成对出现的，他们是一种对称关系：构造分配资源，析
构释放资源；构造用来初始化，析构用来清理。但是现在有一个点他们极不对称：

- 析构通常是虚函数
- 构造不能是虚函数

**构造和析构的另一个不对称的地方是异常处理，构造函数中出错只能抛出异常，但是析
构函数中不允许抛出异常**

## 通常声明成虚函数的析构函数

关于析构通常是虚函数的问题，在 《Effective C++》一书中有讨论，原因很简单：

```cpp
class Base { // 基类公共资源 };
class Sub : public Base { // 子类自己的资源 };
```

为了能够进行多态的处理程序，我们需要通过基类指针指向子类的对象：

```cpp
Base* base = new Sub;
```

那么问题来了，当我们调用

```cpp
delete base;
```

的时候，假如 Base 没有把析构函数声明成虚函数，那么它会直接调用 Base 的析构函数（
这一点在编译的时候就已经确定了），那么 Sub 中分配的子类资源得不到释放这就造成了
资源泄露。如果 Base 的析构函数是虚函数，那么它实际上会调用到子类的析构函数，而子
类的析构函数会自动调用基类的析构函数，最终使得基类和子类的资源都会得到释放。

```cpp
Sub::~Sub() {
    this->Base::~Base(); // 编译器安插代码。
}
```

## 不能声明成虚函数的构造函数

构造函数不能是虚函数的原因很简单，这是一个鸡生蛋，蛋生鸡的问题。要让虚函数正常的
工作，关键在于每个对象中都存在的 vptr 指向正确的虚表。但是这个 vptr 不会凭空出现
并已经设置好了的，它总得在某个地方设置好，而这个设置点最合理的位置就是构造函数。
如果把构造函数声明成虚函数，那么谁来正确的处理vptr呢？所以我们不能把构造函数声明
成析构函数。

# 构造函数中无法正常的调用虚函数

给出下面的代码：

```cpp
class Base {
public:
    Base() {
        name();    // ***1***
    }

    void name() {
        vname();
    }

    virtual char* vname() {
        return "Base";
    }
};

class Drived : public Base {
public:
    virtual char* vname() {
        return "Drived";
    }
};

Base* base = new Drived;
```

上面程序中调用的 vname() 是 Base 中的 vname 实体而不是 Drived 中的 vname() 实体
，因为实际上编译器处理过的构造函数的伪代码如下：

```cpp
Base::Base() {
    __vptr = __base_vtbl; // 编译器安插的代码

    name();
}

Drived::Drived() { // 编译器合成并安插的代码
    Base::Base();

    __vptr = __drived_vtbl;
}
```

也就是说设置合适的 vptr 是在调用基类的构造函数之后，基类的构造函数中调用的虚函数
即使是转换成了虚表中的函数调用，也不可能调用到子类的中函数实现，因为那时候 vtpr
还没有指向子类的虚表。关于代码的安插顺序可以参考《深度探索C++对象模型》一书。

## 如何实现虚构造函数

嗯，这个问题有点白痴。最直接的答案是：不可能，不那么直接的答案是：工厂方法。工厂
方法模式的别名就是虚构造函数，因为我们没有办法把构造函数声明成虚拟函数，所以我们
只有通过曲线救国的方式：通过工厂方法来返回可由子类重新定义类型的实例。

```cpp
class Widget { };
class Button : public Widget { };
class Checkbox : public Widget { };

class WidgetCreator {
public:
    virtual Widget* CreateWidget() {
        return new Widget;
    }
};

class CheckboxCreator {
public:
    virtual Widget* CreateWidget() {
        return new Checkbox;
    }
};

class ButtonCreator : public WidgetCreator {
public:
    virtual Widget* CreateWidget() {
        return new Button;
    }
};
```

用户不再直接通过调用 Widget 来获得 Widget 或者它子类的实例，而是通过上面这样的工
厂方法`CreateWidget` 获得。

```cpp
WidgetCreator* creator = CheckboxCreator::Instance();
Widget* widget = creator.CreateWidget(); // 返回 Checkbox

creator = ButtonCreator::Instance();
Widget* button = creator.CreateWidget(); // 返回 Button
```

因为这个方法可以重载同时又提供构造函数的功能，所以它也被成为虚拟构造函数。关于工
厂方法的详细信息可以查看《设计模式》一书。

