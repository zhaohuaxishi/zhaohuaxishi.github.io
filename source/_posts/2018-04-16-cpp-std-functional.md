---
title: C++轮子——函数式编程
date: 2018-04-16 09:30:44
tags:
 - C/CPP
---

和算法配套出现的组件除了迭代器之外还有仿函数，这篇文章会重点介绍仿函数的使用以及和它相关的函数式编程工具。

# 仿函数

算法库中有很多算法都有一个重载的版本，接收一个`Callable object`用于提升算法的灵活性。

```C++
template< class InputIt, class UnaryPredicate >
InputIt find_if( InputIt first, InputIt last,
                 UnaryPredicate p );

template< class InputIt, class T, class BinaryOperation >
T accumulate( InputIt first, InputIt last, T init,
              BinaryOperation op );
```

语法上可调用对象（`Callable object`）是指可以使用函数调用符号操作它的任何对象，它可以表示很多对象，比如普通的函数，也比如一个重载了函数调用符的类型对象。如果可调用对象的返回值是`bool`则称之为`Predicate`（有人翻译成谓词）。接收`Predicate`的算法通常都以`_if`结尾，比如：`std::find_if`，`std::copy_if`。

我们把重载了函数调用符的对象称为仿函数，它在功能上是一个函数，在语法上是一个类。它和普通函数最大的区别是它可以保存内部状态。

假如我们现在要找出第一个奇数，我们可以这样写：

```C++
bool is_odd(int a) { return a % 2; }
std::find_if(std::begin(a), std::end(a), is_odd)
```

假如我们要查找第一个偶数，我们可以写成这样：

```C++
bool is_even(int a) { return !is_odd(a); }
std::find_if(std::begin(a), std::end(a), is_even)
```

但是如果我实现成仿函数，我们可以这样写：

```C++
class FindOdd {
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

std::find_if(std::begin(a), std::end(a), FindOdd(true));
std::find_if(std::begin(a), std::end(a), FindOdd(false));
```

上面这个例子其实并不太恰当，我不推荐在一个函数中实现两个功能，但是它展示了仿函数区别于普通函数的重要特性——可以保存状态。

<!--more-->

# Predicate 应该是纯函数

`Predicate`是指返回值为`bool`的`Callable object`，而纯函数是指这个对象不会有`side effect`，也就是说如果以同样的参数调用这个对象，结果是一样的。之所以这么建议是因为`Predicate`作为算法的参数而存在，而参数的传递是值拷贝，如果`Predicate`不是纯函数，而算法内部实现中拷贝的这个`Predicate`，它会导致`Predicate`的失效。比如我们先要删除容器中的第三个元素，我们可以写一个仿函数如下：

```C++
class ThirdElement {
public:
    ThirdElement() : called_times_(0) {}

    bool operator()(int a) {
        return ++called_times_ == 3;
    }
private:
    std::size_t called_times_;
};
```

然后用`remove_if`算法：

```C++
std::vector<int> ia = {1, 2, 3, 4, 5, 6};
ia.erase(std::remove_if(std::begin(ia), std::end(ia), ThirdElement()));
```

可惜的是，这段代码可能把第六个元素也给删除掉了，因为`remove_if`可能实现成这个样子。

```C++
template <typename FwdIterator, typename Predicate>
FwdIterator remove_if(FwdIterator begin, FwdIterator end, Predicate p) {
    begin = find_if(begin, end, p);
    if (begin == end)
        return begin;

    FwdIterator next = begin;
    return remove_copy_if(++next, end. begin, p);
}
```

这里的主要问题是`p`，这个变量在`find_if`中拷贝了一份，在`remove_copy_if`中又拷贝了一份。而这两份是独立的对象，状态不共享，所以删除了两次。

当然，不要天真的认为把`called_times_`设置为static，共享状态就没事儿了，因为它会导致你第一次调用可行，第二次调用却会失败。

# lambda

很多时候，我们为算法提供的`Callable Object`是一个非常简单的函数，比如前面提到的`is_odd`和`is_even`。单独处于这个目的创建一个函数或者仿函数其实比较繁琐。C++11中加入的`lambda`可以很好的解决这个问题。查找奇数，我们可以这样写：

```C++
std::find_if(std::begin(ia), std::end(ia), [](int a) {
    return a % 2;
});
```

对于短小的函数，这种写法会更简单，也更优雅。

## 闭包

和绝大部分语言中的`lambda`一样，它可以用来创建闭包。所谓闭包是指一个函数以及相关的引用环境组合而成的实体。比如在Python里面一个闭包可以这样写：

```Python
def print_zen_of_python():
    msg = "zen of python"
    def printer():
        print(msg)
    return printer

closure = print_zen_of_python()
closure()
```

这里面`print_zen_of_python`内部的`printer`函数以及引用环境`msg`组成来一个闭包。这个概念在C++中也是一样的，不同的是，C++不存在垃圾的自动回收，所以环境的捕获需要自己手动完成。捕获方式是把要捕获的变量放在方括号中，比如：

```C++
int num = 2;
std::find_if(std::begin(ia), std::end(ia), [num](int a) {
    return a % num;
});
```

变量的捕获默认都是以拷贝的方式放到`lambda`对象中，你也可以使用引用来捕获变量：

```C++
int num = 2;
std::find_if(std::begin(ia), std::end(ia), [&num](int a) {
    return a % num;
});
```

如果在成员方法中，你可以使用捕获当前对象（这种方式实际上是使用引用捕获`*this`）。

```C++
std::find_if(std::begin(ia), std::end(ia), [this](int a) {
    return a % num_;
});
```
`lambda`更多的语法细节不在这里讨论，有兴趣的可以参考[Lambda expressions](http://devdocs.io/cpp/language/lambda)。

## 匿名的函数对象

实际上，`lambda`构造的是一个匿名的函数对象，比如：

```C++
[&num](int a) {
    return a % num;
}
```

实际上等同于（编译器的具体实现应该不是下面这个样子，构造函数应该是不需要的，具体实现方式我清楚，但是逻辑上来说两者等价）：

```C++
class ClosureType {
public:
    ClosureType(int& num) : num_(num) {}

    auto operator()(int a) const {
        return a % num_;
    }
private:
    int& num_;
};
```

注意，默认情况下，函数调用符重载用默认用的`const`，这意味着你不能改变通过值拷贝捕获到`lambda`内部的值，下面的写法是不合法的：

```C++
[num](int a) {
    num++;  // error
    return a % num;
}
```

原因我们在上文中有提到，这里不在赘述。如果需要改变内部值，可以在参数列表后面加上`mutable`：

```C++
[num](int a) mutable {
    num++;  // ok
    return a % num;
}
```

此外，函数的返回值可以默认推导，如果你需要指定返回值（比如自动推导会失败），可以使用后置返回值的方式：

```C++
[num](int a) mutable -> bool {
    num++;  // ok
    return a % num;
}
```

C++14之后，参数的类型可以是auto，捕获列表可以有初始化值：

```C++
[num](auto a) {
    return a % num;
}

[num = this->num_](auto a) {
    return a % num;
}
```

# std::function

在C++中，可调用对象包括很多种，比如普通函数、函数指针、成员函数指针、仿函数、lambda。对于这些可调用对象C++11提供来一个高阶的封装：`std::function`，它可以用于表示各种各样的可调用对象（甚至包括读取成员变量），下面这个例子来自`cppreference`。

```C++
// store a free function
std::function<void(int)> f_display = print_num;
f_display(-9);

// store a lambda
std::function<void()> f_display_42 = []() { print_num(42); };
f_display_42();

// store the result of a call to std::bind
std::function<void()> f_display_31337 = std::bind(print_num, 31337);
f_display_31337();

// store a call to a member function
std::function<void(const Foo&, int)> f_add_display = &Foo::print_add;
const Foo foo(314159);
f_add_display(foo, 1);

// store a call to a data member accessor
std::function<int(Foo const&)> f_num = &Foo::num_;
std::cout << "num_: " << f_num(foo) << '\n';

// store a call to a member function and object
using std::placeholders::_1;
std::function<void(int)> f_add_display2 = std::bind( &Foo::print_add, foo, _1 );
f_add_display2(2);

// store a call to a member function and object ptr
std::function<void(int)> f_add_display3 = std::bind( &Foo::print_add, &foo, _1 );
f_add_display3(3);

// store a call to a function object
std::function<void(int)> f_display_obj = PrintNum();
f_display_obj(18);
```

## 回调的处理

这个模板类可以用于非常方便的处理回调函数。

在`std::function`出现之前，回调可能使用动多态来实现的：

```C++
using Renderer = std::shared<IRenderer>;

Renderer renderer = CreateRenderer();
void HandleData(const Data& data) {
    auto frame = Decode(data);
    renderer->Render(frame);
}
```

这种方式使得所有的渲染类需要继承子IRenderer，这是继承的解决方案。另外一种方案是使用`std::function`把继承变成组合。

```C++
using Renderer = std::function<void(const Frame&)>;

auto render = std::bind(&RendererImpl::Render, renderer)
void HandleData(const Data& data) {
    auto frame = Decode(data);
    render(frame);
}
```

详细的用法，请参考[std::function](http://devdocs.io/cpp/utility/functional/function)。

## 如何多态的存储一个函数对象

`std::function`的实现，在《C++设计新思维》一书中第五章有非常详细的介绍，有兴趣的可以参考这本书。

# std::bind

前面谈到，`std::function`是`Callable Object`的封装，而`std::bind`可以认为是`Callable Object`的Adaptor，它可以用于绑定`Callable Object`的部分参数，从而变成改变函数调用的接口。比如：

```C++
using Foo = std::function<int(int)>;

int Bar(int a, int b) {
    return  a * b;
}

Foo double_num = std::bind(Foo, 2, std::placeholder::_1);
assert(double_num(3) == 6);
```

这里面最有意思的地方在于，`std::bind`把`Bar`这个需要两个参数的函数变成了一个只需要一个参数就能调用的对象。这个函数可以非常方便的把一个成员函数转换成一个普通方式就可以调用的函数，这在创建线程，注册回调的时候非常方便：

```C++
auto feature = std::async(std::bind(&Renderer::Render, this));
asio::async_read(socket, buffer, std::bind(&DataReceiver::OnReadData, this));
```

当然，`std::async`自身可以处理成员函数的情况，所以上面例子中的`std::bind`是多余的。

## 实现细节

`std::bind`最精妙的地方在于它可以使用`std::placeholder::_1`这样的占位符来实现实参的自动分派，它内部的实现逻辑极其精妙，我曾经单独写过[一篇文章](/2017/01/27/bind-implementation/)简要的分析它的实现逻辑，有兴趣的同学可以参考一下，这里不在赘述。

---

【1】如果成员变量是引用，`const`成员函数里面可以修改引用指向的数据的值。这个问题换成指针可能会好理解一点，`int * const i` 表示你不能修改i的值，而不是不能修改i指向的值。引用一旦绑定就不可能更改，所以成员函数`const`与否对于引用来说意义不大。
