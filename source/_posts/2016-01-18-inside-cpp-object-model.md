title: 读书札记之 —— 《深度探索 C++ 对象模型》
date: 2016-01-18 11:30:31
tags:
 - 读书笔记
 - C/CPP
---

这本书应该有一个副标题——隐藏在编译器背后的故事。这是一本非常有深度的书，它告诉我
们编译器在实现C++的那些特性（封装，单继承，多继承，虚拟继承、虚函数，构造，拷贝
，析构，模板，内联，异常，RTTI等等）时帮我们做了什么。

译者侯捷在译序中说的本立道生可谓是一语中的。不了解这些东西你可以写出正确的C++程
序，因为你知道C++语法。但是如果你知道C++中的特性是如何实现的，它们的开销是什么，
你就可以在单继承、多继承、虚拟继承之间；抽象类、实体类之间；继承和组合之间做出最
合适你自己的选择。同时了解这些知识会让你对于自己写的程序理解的非常透侧，一个优秀
的程序员要对自己写下的代码了如指掌。

<!--more-->

# 关于书中的错误

这本书原版就有很多瑕疵，书中有很多编辑上的错误，译者侯捷先生在翻译的时候对大部分
的错误进行了更正，所以中文译本在编辑上的错误比较少，这一点非常感谢译者。

不过这本书的翻译比较诡异，大多数地方都是中英文混合，侯捷先生说是为了让大家熟悉英
文的专业术语。总体来说，这种翻译风格并不会给阅读带来太大的影响，毕竟长期阅读计算
机专著的人对于这些术语也不是很陌生。

不过本书有一定的难度，翻译起来比较困难，侯老爷子虽然功力非常的深厚，但是翻译也难
免会出现错误。我在阅读的过程中凡是遇到不通顺或者难以理解的地方基本上都会参考英文
的原文辅助理解。这个过程中我发现其实本书还是有很多地方的翻译是有待商榷的，我把我
找到的这些地方整理成了另一篇博文——[细数《深度探索C++对象模型》一书中的20个翻译错
误][icom-translation-error]。当然这些都是个人对于原文的理解，有可能是我理解错误
，大家可以看一看自行判断。

 [icom-translation-error]: /2016/01/18/icom-translation-error/

# 知识点小结

书中对于各种特性的讲解的非常的仔细，此处对书中的一些信息按照特性分类总结。

## 封装

所谓封装就是把数据和处理数据的方法当入到同一个结构中，书中使用 ADT（抽象数据类型
） 一词来表示纯粹的封装。

```cpp
class ADT {
public:
    void fun(int arg);
private:
    int data;
};
```

C++把函数放到对象之外，类的内部只是存放非静态的数据。所以上面的封装最终会转换成
类似下面的C代码。

```cpp
struct ADT {
    int data;
};

void ADT_fun(ADT* this, int arg);
```

如果你要使用C语言来模拟封装，完全可以用上面这种方式模拟。

## 单继承

单纯的单继承不会带来任何的开销，因为下面的继承：

```cpp
class Base { public: int foo; };
class Sub : public Base { public: int bar; };
```

和下面的结构体：

```cpp
struct Sub {
    int foo;
    int bar;
};
```

得到的内存布局是一样的。其实你甚至可以自己用C语言模拟这种继承：

```cpp
struct Base {
    int foo;
};

struct Sub {
    struct Base base;
#define foo base.foo
    int bar;
};
```

在Linux内核的网络协议栈中有大量的这种写法。

## 多继承

多继承的主要问题在于处理第二个和之后的基类时，需要调整偏移量，以便能找到正确的地
址，比如有如下继承体系的情况下。

```cpp
class Sub : public FirstBase, public SecondBase { ... };
Sub* sub = new Sub;
```

那么 Sub 和 FirstBase 直接可以直接转换，编译器不需要额外的处理。

```cpp
FirstBase* fb = sub;
```

但是 Sub 和 SecondBase 之间的转换需要编译器调整偏移量，给定下面的代码：。

```cpp
SecondBase* sb = sub
```

编译器如果要保证正确性需要把它转换成：

```cpp
SecondBase* sb = (SecondBase*) ((char*) sub + sizeof(FirstBase));
```

这主要是因为在一个完整的子类对象中，以此存放了各个基类的子对象（P112）。

## 虚拟继承

虚拟继承是最蛋疼的问题，因为它意味着共享，给定下面的继承体系。

```cpp
class VBase { public: int a; };
class RBase1 : public VBase { };
class RBase2 : public VBase { };
class Sub : public RBase1, public RBase2 { }
```

下面的代码中，编译器无法之道 a 到底是通过 RBase1 继承而来的还是通过 RBase2 继承
而来的，所以这段代码无法通过编译。

```cpp
Sub sub;
sub.a = 10;
```

解决上面这种菱形继承的最好的方式就是使用虚拟继承，


```cpp
class RBase1 : public virtual VBase { };
class RBase2 : public virtual VBase { };
```

这样一来体系中就会共享同一个 VBase 的子对象。但是代价是非常高的，因为整个对象的
内存模型都会编码，VBase 不再作为 RBase1 子对象。所以原本适用于这个模型的所有东西
基本上都不再使用。访问 VBase 子对象现在需要通过一个额外的变量（可以是指针，也可
以是偏移量）访问，效率上要比普通的继承低。

此外为了实现共享，虚拟基类由继承体系中最底端的类负责构造而不是直接子类负责构造，
比如：

```cpp
Sub sub;
```

`sub` 对象的构造过程中，它需要负责调用 RBase1，RBase2（实体基类）和 VBase（虚拟
基类）的构造函数，但是 RBase1，RBase2 的构造函数不能调用 VBase 构造函数。这种机
制非常的复杂，需要通过给构造函数添加额外的参数来完成

```cpp
Sub::Sub(Sub* this, bool __most_drived) {
    if (__most_drived) {
        this->VBase::VBase();
    }

    this->RBase1::RBase1(false);
    this->RBase2::RBase2(false);
    ...
}

RBase1::RBase1(RBase1* this, bool __most_drived) {
    if (__most_drived) {
        this->VBase::VBase();
    }
    ...
}

RBase1::RBase1(RBase1* this, bool __most_drived) {
    if (__most_drived) {
        this->VBase::VBase();
    }
    ...
}
```

编译器在对象创建的时候传递合适的 `__most_drived`。

```cpp
// Sub sub;
Sub::Sub(&sub, true);
```

当涉及到虚拟继承的时候，几乎所有的机制都会变得非常的复杂，所以不用为好。

## 虚函数

### 单继承下

虚函数的引入使得每一个函数虚函数的类都必须包含对于的虚表（vtbl——里面存放实际调用
的函数的地址），而每一个对象都有虚指针（vptr）指向正确的虚表。给定下面的继承体系
。

```cpp
class Base {
public:
    virtual char* foo() {
        return "foo";
    }
};

class Sub {
public:
    virtual char* foo() {
        return "foobar";
    }
};
```

相应的客户端代码如下：


```cpp
Base* pbase = new Sub;
cout << pbase->foo() << endl; // "foobar"
```


上面的代码之所以能够正常的工作的原因是，编译器把 `pbase->foo()` 的调用改成了对于
虚表中的函数的调用：

```cpp
(*pbase->__vptr[1])(pbase); // p147
```

### 多继承下

多继承下如果涉及到第二个基类，那么需要编译器调整 this 指针的位置，给出如下的继承
体系：

```cpp
class FirstBase { };

class SecondBase {
public:
    virtual char* foo() { return "foo"; }
};

class Sub : public FirstBase, public SecondBase {
public:
    virtual char* foo() { return "foobar"; }
};

```cpp

如果要下面的代码能够得到正确的处理，编译器需要做特殊的处理。

```cpp
SecondBase* sb = new Sub;
sb->foo(); // "foobar"
```

如前面继承所以，子类和第二个基类之间的转换成需要加上偏移量，以得到争取的基类地址
。此时因为多台 sb 所指向的子对象对应的 vtbl 中存放的方法是 Sub 中的成员方法，直
接进行前面提到的转换：


```cpp
(*sb->__vptr[1])(sb);
```

并不能得到正确的结果，因为 Sub 的 `foo` 成员函数经过编译器的处理会变成这样：

```cpp
char* foo(Sub* this); // 为了清晰性，没有加入 name mangling
```

但是此时 `sb` 指向的却是一个 `SecondBase` 子对象，所以为了能够得到正确的函数调用
，必须对`sb`进行修正加上合适的偏移量使它指向 `Sub` 对象的地址。这个修正同样比较
复杂，书中提到的做法是通过thunk，或者改变vtabl中实体的类型，让他不再只是存储函数
的地址，同时也存储this的偏移量。（后面这种做法对于那些不需要调整this指针的函数来
说效率太低）thunk是包含了偏移量设置和实际函数调用的代码块，它的作用有点类似于设
计模式中的装饰者模式，通常thunk看上去像下面这个样子。

```cpp
sb_foo_thunk:
    this += sizeof(FirstBase);
    Sub::foo(this);
```

## 构造

构造函数的主要问题是在需要的时候它必须合成，或者在已有的函数上安插代码，下面是常
见的几种情况。

### 组合

```cpp
class Widget {
private:
    String str;
    int val;
};
```

对于类编译器需要合成一个构造函数用来安插 `str` 成员变量的构造。

```cpp
Widget::Widget() { // 合成的构造函数
    str.String::String();
}
```

### 继承

可能大部分人认为只要有继承就一定会合成默认构造函数以调用基类的构造函数，但是这种
观点是错误的，编译器只会在必要的时候这么做。

```cpp
class Base { public: int i; }
class Sub : public Base { public: int i; }
```

对于上面的继承，编译器不会合成默认构造函数，因为构造函数被判定为 trivial（不重要
）。但是像下面这种代码则会合成构造函数：

```cpp
class Base { public: String str; }
class Sub : public Base { public: int i; }
```

### 虚函数

如果要处理虚函数就必须要设置好 vptr 指向正确的 vtbl，所以只要有有虚拟函数的定义
就一定会有代码的安插，无论是在合成的构造函数中还是已有的构造函数中。比如下面的类
定义。

```cpp
class Widget {
public:
    virtual char* foo();
};
```

## 其他

书中还是非常多其他的内容，包括内联、异常、RTTI、模板等等。此处不再总结，这本书非
常值得一读，推荐所有有一定基础的C++程序员阅读这本书。
