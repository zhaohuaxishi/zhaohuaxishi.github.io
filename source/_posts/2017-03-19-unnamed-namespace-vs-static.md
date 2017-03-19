---
title: 匿名命名空间和 static 声明的区别
date: 2017-03-19 12:58:26
tags:
 - C/CPP
---

在`C`语言中，如果我们想要一个符号只在文件内部（严格来说是编译单元内部，也就是经
过预处理之后得到的源文件）可用，我们需要把它声明成`static`，比如：

```c
// widget.cpp
static void internal_function() { }
static int internal_value = 10;
```

在`C++`中有一种推荐的等效做法——匿名的名字空间：

``` c++
// widget.cpp
namespace {

void internal_function() {}
int internal_value = 10;

}   // namespace
```

之所以推荐在`C++`使用匿名名字空间来取代`static`的这种用法的原因主要有下面三个：

1. `static`的这个关键词的用途过多，比如你还可以用它声明静态成员，用它声明函数内
   部的静态变量。

2. `static`没有办法修饰一个类型，所以下面的代码不合法：

    ```c
    static struct Widget {}; // 不合法
    ```

    但是匿名命名空间可以：

    ```c++
    namespace {
    struct Widget {};   // 合法
    }   // namespace
    ```

3. 某些模板的参数必须具有外部链接熟悉，比如下面的代码是不合法的

   ```c++
   template <typename T, const int& N>
   struct Array {
       T data[N];
   };

   static const int kMaxSize = 10;

   Array<int, kMaxSize> data;   // 不合法，报错说 kMaxSize 没有外部的链接属性
   ```

虽然`static`关键字和`匿名命名空间`完成了类似是事情，但是这两者之间到底有什么本质
的区别呢？这篇文章的剩余部分会详细说明这个问题。

<!--more-->

# 本质区别

我想这两者的本质区别应该在于，他们声明出来的符号的链接属性不同。`static`关键字声
明的符号有`内部链接属性`而，匿名命名空间中声明的符号有`外部链接属性`。

## static

`C`和`C++`中每一个源文件（.c, .cpp）都可以单独编译成一个目标文件(.o)，之后通过链
接器把这些目标文件链接起来，形成最后的可执行文件或者库文件。也就是说某个源文件
`a.cpp`中用到的符号`s`（函数，全局变量等）可能是在另一个源文件`b.cpp`中定义的，
在 `a.cpp`中只要给出相应的声明就可以了。为了让链接器可以找到符号`s`的定义，`b.o`
必须提供它定义的所有可链接的符号。`b.o`只会提供具有外部链接熟悉的符号给链接器使
用，如果一个符号在声明中加了`static` 关键字，那么它的链接属性变成了内部链接，也
就不会暴露给链接器进行链接，这样一来它也就只能被文件内部看见了。

## 匿名名字空间

匿名名字空间并不是真的没有名字，只不过这个名字只有编译器知道而已，下面的代码

```c++
namespace {
struct Widget {};
} // namespace
```

实际上经过编译器的处理之后可能是下面这个样子：

```c++
namespace some_unique_name_compiler_generated {}    // 编译器生成一个唯一的名字
using namespace some_unique_name_compiler_generated;

namespace some_unique_name_compiler_generated {
struct Widget {};
}
```

所以实际上 `Widget` 类完整的修饰应该是：

```c++
some_unique_name_compiler_generated::Widget;
```

因为它没有 `static` 的修饰，所以它具有外部连接，但是因为命名空间的名字是编译器生
成的，外部无从知晓，所以`Widget`这个符号只有编译单元的内部才能看见。
