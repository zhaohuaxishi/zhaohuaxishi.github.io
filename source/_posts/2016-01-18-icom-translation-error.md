title: 细数《深度探索C++对象模型》一书中的20个翻译错误
date: 2016-01-18 17:49:00
tags:
 - C/CPP
---

《深度探索C++对象模型》一书由侯捷先生翻译，侯先生是非常有名的C++专著译/作者，这
本书可以看得出来他也花了不少心思翻译。但是我在阅读的过程中还是发现了一些翻译上的
不足。我在下笔写这篇文章的时候纠结到底是用瑕疵一词还是错误一词呢？我最终还是选择
了错误一词，因为有些地方确实有误导之嫌疑，我对照英文原文和中文译本，把我认为可能
是翻译错误的地方整理在此文中。这些地方有可能是我误解了作者的意思而侯捷理解正确了
，也有可能是我本身误解了侯捷的翻译，我把原文和译文一并贴出，读者可以自行对照判断
。

**此文中的页码和行号都是简体译本中的页码和行号，行号中的 - 表示倒数，行号不包括
代码内容，行号前面的码表示是代码的错误**

<!--more-->

# P46，L-4

译文如下：

> 原先cfront的做法是靠“在derived class object 的每一个virtual base classes
> 中都安插一个指针”完成。

原文如下：

> In the original cfront implementation, for example, this is accomplished
> by inserting a pointer to each of the virtual base classes within the derived
> class object.

这一个错误非常明显，原文中说 “inserting a pointer ... within the derived class
object.“ 意思是说在子类中安插指向每一个虚拟基类的指针，而不是在虚拟基类中安插指
针。

这段文字后面的代码可以证明：

```cpp
void foo(const A* pa) { pa->__vbcX->i = 1024; }
```

pa 的类型是 A，它是虚拟基类 X 的子类，指针安插在它的对象上面而不是安插在虚拟基类
X 的对象上。此外指针的名字 `__vbcX` 表示指向虚拟基类 X 的指针再一次证明这一点。

# P78，L3

译文如下：

> 我建议你总是把一个member的初始化操作和另一个放在一起（如果你真的觉得有必要的话
> ），放在constructor之内，像下面这样：

原文如下：

> I recommend always placing the initialization of one member with another (if
> you really feel it is necessary) within the body of the constructor, as
> follows:

这句的主干是”placing the initialization of ... with in the body of constructor“
。也就是说”把...的初始化放在构造函数体内“，这一点侯捷的译文没有错，但是”the
initialization of one member with another“的意思应该理解成”使用一个成员来初始化
另一个成员的操作“，而不是把两个成员初始化放在一起。

之所以这么说原因很简单，下面给出的例子代码明显只是把其中一个放入构造函数体内。而
且这样理解明显更加合理，因为这种方式在效率性和正确性上都有比较号的权衡。把独立的
成员变量初始化放在成员变量初始化列表可以提高效率，而把相关的成员编码放入到构造函
数中可以保证正确性。

# P80,L2

上面的那个翻译错误，直接导致了这个翻译错误：译文如下：

> 是因为我要给你一个忠告：请使用”存在于constructor体内的一个member“，而不要使用”
> 存在于member initilazation list 中的member“来为另一个member设置初值。

原文如下：

>  I reiterate my advice to initialize one member with another inside the
>  constructor body, not in the member initialization list.

译文不但读起来很别扭，而且意思也改变了。原文中使用了 `reiterate` 一词表示重申，
他要表达的应该是前面说的把使用一个成员来初始化另一个成员这样的操作放在构造函数体
内。前面说的成员变量而这里说的是成员方法，其实本质上都是一样，因为 xfoo() 可能会
访问到某些没有初始化的成员。

# P90，L5

译文如下：

> 其效果是，如果一个inline函数在class声明之后立刻被定义的话，那么就还是对其评估
> 求值。

原文如下：

> the effect is still to evaluate the body of an inline member function as if it
> had been defined immediately following the class declaration.

这句话错的更加离谱了，按照字面来理解也应该是说，”效果是依然像内联成员函数被立即
定义在类声明之后一样评估求值它“。最明显的证据就是原文中的`still`其实作者想表达的
是效果和P89页上的第2点措施是一样的。

# P106，码L1-L2

译本代码如下：

```cpp
Concrete2*pc2;
Concrete1*pc1_1, *pc1_2;
pc1_1 = pc2;  // 译注：令pc1_1 指向 Concrete2 对象
*pc1_2 = *pc1_1;
```

原文代码如下（原文的代码和译文代码命名不一致，为了方便，我统一改成了译文中的
pc1_2）：

```cpp
Concrete2 *pc2;
Concrete1 *pc1_1, *pc1_2;
pc1_1 = pc2;
*pc1_1 = *pc1_2;
```

也就是说代码的最后一句的赋值是反的。作者的注释和译注的注释都解释了赋值的错误，子
类的子对象备覆盖了，现在 bit2 member 有一个非预期的数值。

那么问题是有子对象的应该是子类，也就是 pc2 所指的对象才对，现在 pc1_1 指向 pc2，
所以被覆盖的应该是 `pc1_1` 而不是 `pc1_2`。所以原文中的代码是没有错误的，但是译
文中的代码有误。可以把译文中的`pc1_1 = pc2`改成`pc1_2 = pc2`以得到正确的结果。

# P149，L1

译文如下：

> 我想你很少看到下面这种怪异的写法：

原文如下：

> it was not uncommon to see the following admittedly bizarre idiom in the code
> of advanced users:

除去后面那些什么高阶用户不说，原文使用了 `not uncommon` 是双重否定应该是表示经常
看到这种代码才对而不是很少，所以此处应该是误译。

# P207，L6

译文如下：

> 如果 class object 是最底层的 class，其 constructors 可能被调用；某些用以支持这
> 一行为的机制必须被放进来。

原文如下：

> These constructors, however, may be invoked if, and only if, the class object
> represents the "most-derived class." Some mechanism supporting this must be
> put into place.

原文中的能且仅能没有翻译，然而所有最大的错误还是在于 These 的翻译，它指的不是最
底层的 class 的构造函数而是基类的构造函数，因为最底层的类必须负责虚拟基类的构造
，而这种构造需要额外的机制支持。

# P219，L2

译文如下：

> 何时需要供应参数给一个 base class constructor？这种情况下在“class 的
> constructor 的 member initialization list 中”调用该class的虚拟函数，仍然安全吗
> ？

原文如下：

> What about when providing an argument for a base class constructor? Is it
> still physically safe to invoke a virtual function of the class within its
> constructor's member initialization list?

这里的的错误非常明显，作者想要表达的意思其实是：

```cpp
class Base {
public:
    void Base(int arg) { }
};

class Drived : public Base {
public:
    void Drived() : Base(get_argument_by_virtual_function()) { }
};
```

作者在前文中说过，在 member initialization list 中调用该类的虚函数初始化成员变量
是安全，所以这里反问“如果在 member initialization list 中给基类构造函数提供参数
时调用该类的虚拟函数是否安全，答案是否定的，原文中的翻译相差太远了。

# P220，L-2

译文如下：

> C++ Standard 上说 copy assignment operators 并不表示 bitwise copy semantics 是
> nontrivial

原文如下：

> The Standard speaks of copy assignment operators' not exhibiting bitwise copy
> semantics as nontrivial

这一句的翻译就更不靠谱了，完全变味了。按原文的理解，这句话应该译文：标准规定不具
备`bitwise copy semantics`的`copy assignment operators`是`nontrivial`。

# P223，L-9

译文如下：

> 然而我们无法支持它，我们仍需要根据其独特的继承体系，安插任何可能个数的参数给
> copy assignment operator

原文如下：

> We can't reasonably support this, however, and still insert an arbitrary
> number of arguments to the copy assignment operator based on the peculiar
> characteristics of its inheritance hierarchy.

很明显，这里作者想要表示的是无法在支持上面的操作的同时给根据其独特的继承体系安插
任何可能个数的参数给 copy assignment operator。所谓安插参数就是像在涉及到虚拟基
类的构造函数一样，通过添加 `__most_drived` 这样的参数来控制那些地方应该调用虚拟
基类的 copy assignment operator。

原文之所以说无法在支持上面的操作（也就是通过指针调用函数）的同时支持安插参数的原
因书中没有说，我也不太理解。但是作者想要表达的应该就是这个问题，文中提到的 6.2
节也讨论过这个问题，因为构造函数是通过指针传递的，所以没有办法设置默认参数。

个人的理解是指针的间接引用是一个运行期的求值问题，安插参数却需要在编译时完成。我
们总不至于得让用户提供我们额外安插的参数。

# P234，L1

译文如下：

> 如果 object 内含一个 vptr，那么首先重设相关的 virtual table。

原文如下：

> If the object contains a vptr, it is reset to the virtual table associated
> with the class.

原文意思是把 vptr 重置以便指向相关的类虚表，而不是重设虚表。

# P235，L1

译文如下：

> 译注：上页的destrucotr扩展形式似乎应为2,3,1,4,5，才符合constructor的相反顺序。

这段译注有误导的嫌疑，个人认为作者的顺序是没有错的，比如给点下面的继承体系：

    A
    ^
    |
    B
    ^
    |
    C

并有如下代码：

```cpp
A* pa = new C;
delete pa;
```

因为析构的顺序和构造是相反的，所以先调用C的析构，此时vptr指向的是C的虚表，所以不
需要重置，调用完C的析构的时候调用B的析构，因为C已经析构完成，相当于目前是一个完
整的B类型了。B的析构函数本体执行之前当然应该首先重置vptr让它指向B的虚表，否则在
析构中调用的函数不就会调用到C的实体中去了吗，所以第一步是（1）不是（2）是合理的
。译者可能是受到和`constructor`相反的顺序这句话的影响给出的译注。

# P252，L6

此处不能说是译者的误译，因为译者说没有充分的理解这句话的意思，所以给出了原文如下
：

> This has always resulted in less than first class handling of the
> initialization of an array of class objects.

我找到的原文如下：

> This has always resulted in less that first-class handling of the
> initialization of an array of class objects.

我得到的原本和译本有一个细微的差别就是 `less that` 而不是 `less then`。这句话的
意思个人理解是，因为构造函数是通过指针调用的，所以没有办法给出非常优雅的方式处理
类对象数组的初始化。关于指针调用函数是无法设置参数的问题在copy assignment
operators 的讨论中也有提到过。

其实作者想要表达的意思应该是没有很合理的方式解决初始化问题，比如后文中给出两种方
式：第一，不允许；第二产生内部的违背语言规则的`stub constructor`，都不是优雅的解
决方式。

# P258，L-7

译文如下：

> 还记得吗？在个别数组元素构造过程中，如果发生 exception，destructor 就会被传递
> 给 vec_new()

原文如下：

> Recall that the destructor is passed to vec_new() in case an exception is
> thrown during construction of the individual array elements.

原文的意思应该是：还记得吗？为了防止在个别数组元素构造过程中发生 exception，
destructor 被传递给了 vec_new()。

# P263，码L2

译文代码如下：

```cpp
Point3d *p = &((Point3d*)ptr)[ ix ] // 译注：原书为 Point *p = ... 恐为笔误
```

原文代码如下：

```cpp
Point *p = &((Point3d*)ptr)[ ix ];
```

其实原文的代码没有错误，因为通过多态的调用，可以正常的析构掉数组元素。否则在后面
的解释中作者不会强调这个调用是虚拟的。

后面的地方之所以要在取下边之前转换成 Point3d* 是为了一次能够偏移 sizeof(Point3d)
的大小而不是 sizeof(Point) 的大小。

# P268，码L-5,L-3

译文代码如下：

```cpp
T temp;
temp.operator+(a, b); //（1） 译注：原书是 a.operator+(temp,b); 误
```

> 标示为（-1）的那一行，未构造的临时对象被赋值给operator+()。

原文如下：

```cpp
T temp;
a.operator+(temp, b); // 1
```
> In the line marked //1, the unconstructed temporary is passed to operator+().

其实后文中作者说的很明显，temp没有构造就直接传递给了operators，然后通过拷贝构造
或者NRV的constructor进行构造。所以这个地方其实是把 temp 当成是一个优化返回值的对
象。译者把它理解成结构的返回值，是对于作者的原本意思的误解。

# P269，L8

译文如下：

> 因此以一连串的 destruction 和 copy construction 来取代 assignment 一般而言是不
> 安全，而且会产生临时对象。

原文如下：

>  Therefore the replacement of assignment with a sequence of destruction and
>  copy construction is generally unsafe and the temporary is generated.

作者在P268中给出的代码中会产生temp变量，P269中的代码用析构和拷贝构造替换之后不会
产生临时变量。

```cpp
// ********* 有临时变量*************
T temp;
a.operator+( temp, b );    // 1

// c = temp
c.operator =( temp ); // 2
temp.T::~T();

// ********* 无临时变量*************
c.T::~T();
c.T::T( a + b );
```

但是这种转换不一定是安全的，因为赋值和先析构再拷贝并不一定就是同样效果，如果效果
不一样，那么就只能通过产生临时变量的方式解决上述问题。译文中的而且会产生临时对象
有误导之嫌。

# P300，L1

译文如下：

> 把两块区域以个别的”备摧毁之local object“链表（已在编译时期设妥）联合起来

原文如下：

> associate the two regions with separate lists of local objects to be destroyed

原文的意思是给两块区域分别于不同的链表关联起来。

# P305，L7，L10

译文如下：

> 相同形式的 exception 数据栈中。

原文如下：

> some form of exception data stack。

译者把 some 看成了 same，应该翻译成：某种形式的 exception 数据栈中。

同一段的最后一句也有错误，译文如下：

> 以及可能会有的 exception object 描述器（如果有人定义它的话）。

原文如下：

> and possibly the address of the destructor for the exception object, if one if
defined.

译者把 destructor 看成了 descriptor，应该翻译成：以及可能会有的析构函数地址。

# P306，L1，L8，L-4

这三个地方都用了”繁殖“一词，但是原文是”propagated“一词，翻译成传播或者传递会比翻
译成繁殖更准确一些。
