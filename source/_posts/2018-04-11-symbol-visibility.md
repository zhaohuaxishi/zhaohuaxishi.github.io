---
title: 一种跨平台的C/C++动态库的符号隐藏方式
date: 2018-04-11 11:31:49
tags:
 - C/C++
---

# 什么是符号隐藏

在同一个文件中，如果有一些函数我们并不想要让外部访问，我们通常会添加 static
修饰符，把它设置为内部链接属性。

```c++
static void foo();
```

但是通常库不太可能是单文件组成，这些文件中有些是做接口给外部使用，有些则单纯的只是库的内部实现。对于外部使用者来说，内部实现的这些符号没有实际的作用，理论上我们完全可以像对待文件内部符号一样把它们统统隐藏掉。但是在语言层面我们并没有相关的语法用于表达这个概念（Java中的包访问权限和C#中的internal类似这个概念）。不同的编译器提供了不同的方式来完成这件事情，这篇文章总结了一种跨平台的处理方式。

<!--more-->

# 符号隐藏的作用

一般来说做符号隐藏有以下三个作用：

- 安全，去掉不必要的符号，可以增加逆向破解的难度。
- 压缩空间，符号实际上是放在 dll 中的，去掉这些符号可以缩减 dll 的大小
- 性能，符号隐藏掉意味着它不会参与到动态链接过程，编译器可以有更大的优化空间，可能会产生更好的性能。

# 如何做符号隐藏

符号隐藏可以采用下面几个步骤（文中假定你使用MSVC或者4.0以上版本的GCC，低版本GCC不支持符号隐藏）：

## 1. 动态库

符号能否隐藏在于它在动态链接的过程中是否需要用到。静态库实际上是目标文件的集合，它并没有完成链接过程。所以符号隐藏通常都是基于动态库的，静态库的符号隐藏没有很好的跨平台方式，如果想要尝试，可以参考下面这些链接。

- [How to apply gcc -fvisibility option to symbols in static libraries?](https://stackoverflow.com/questions/2222162/how-to-apply-gcc-fvisibility-option-to-symbols-in-static-libraries)

- [Symbol hiding in static libraries built with Xcode/gcc](https://stackoverflow.com/questions/3276474/symbol-hiding-in-static-libraries-built-with-xcode-gcc?noredirect=1&lq=1)

## 2. 默认隐藏所有的符号

MSVC和GCC在动态库符号的默认属性上面有较大的差别，MSVC默认所有的符号都是隐藏的，而GCC默认所有的符号都是可见的。虽然我不太喜欢臃肿的MSVC，但是我不得不承认在这一点上，我更倾向于MSVC的选择。

如果你使用`MSVC`编译器，这个步骤你可以什么都不做，如果你使用GCC，你需要给你的编译器加上`-fvisibility=hidden`选项，你也可以加上`-fvisibility-inlines-hidden`把内联函数隐藏掉。如果你使用`Autotool`，你可以通过设置`LD_CXXFLAG`来控制默认隐藏，如果你使用`CMake`，可以通过`set(CMAKE_CXX_VISIBILITY_PRESET hidden)`来完成这一点。

## 3. 把你想要公开的接口的属性设置为外部可见

在MSVC中，我们通过在编译的时候设置`__declspec(dllimport)`和使用的时候设置`__declspec(dllexport)`来完成这一点，在GCC中则简单一些统一设置成`__attribute__ ((visibility ("default")))`即可。

# 辅助宏

上面这些步骤比较繁琐，通常会定义宏来协助处理这一部分内容，下面是来自`GCC WIKI`的一个模板

```c++
// Generic helper definitions for shared library support
#if defined _WIN32 || defined __CYGWIN__
  #define FOX_HELPER_DLL_IMPORT __declspec(dllimport)
  #define FOX_HELPER_DLL_EXPORT __declspec(dllexport)
  #define FOX_HELPER_DLL_LOCAL
#else
  #if __GNUC__ >= 4
    #define FOX_HELPER_DLL_IMPORT __attribute__ ((visibility ("default")))
    #define FOX_HELPER_DLL_EXPORT __attribute__ ((visibility ("default")))
    #define FOX_HELPER_DLL_LOCAL  __attribute__ ((visibility ("hidden")))
  #else
    #define FOX_HELPER_DLL_IMPORT
    #define FOX_HELPER_DLL_EXPORT
    #define FOX_HELPER_DLL_LOCAL
  #endif
#endif

// Now we use the generic helper definitions above to define FOX_API and FOX_LOCAL.
// FOX_API is used for the public API symbols. It either DLL imports or DLL exports (or does nothing for static build)
// FOX_LOCAL is used for non-api symbols.

#ifdef FOX_DLL // defined if FOX is compiled as a DLL
  #ifdef FOX_DLL_EXPORTS // defined if we are building the FOX DLL (instead of using it)
    #define FOX_API FOX_HELPER_DLL_EXPORT
  #else
    #define FOX_API FOX_HELPER_DLL_IMPORT
  #endif // FOX_DLL_EXPORTS
  #define FOX_LOCAL FOX_HELPER_DLL_LOCAL
#else // FOX_DLL is not defined: this means FOX is a static lib.
  #define FX_API
  #define FOX_LOCAL
#endif // FOX_DLL
```

我们想要导出一个符号的时候使用`FOX_API`：

```c++
class FOX_API Fox {};
```

在编译动态库的时候，设置`FOX_DLL`和`FOX_DLL_EXPORTS`这两个宏。在使用动态库的是，定义`FOX_DLL`这个宏。
