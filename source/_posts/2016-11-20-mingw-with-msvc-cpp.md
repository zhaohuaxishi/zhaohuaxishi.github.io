---
title: 在 MSVC 中使用 MinGW 编译的 C++ 库的方式
date: 2016-11-20 18:30:21
tags:
 - C/CPP
---

这篇文章介绍一种能让 MSVC 使用 MinGW 编译的 C++ 库的方式。（这里只讨论C++库的共
享，MinGW编译出来的C库好像使用可以直接在MSVC中使用的。）

<!--more-->

# 为什么使用这种方式

有时候你需要编写跨平台的`back-end`程序，由不同的系统平台去实现`fore-end`。如果使
用 GCC，你可以通过交叉编译得到适用于不同平台的`back-end` 库。为了实现 Windows 的
GCC 交叉编译你需要用到 MinGW，而 Windows 上很多人使用 MSVC 编写程序，包括你的
`fore-end` 程序。这样依赖你就面临了需要让 MSVC 使用 MinGW 编译的库的问题了。

# MinGW 输出的文件类型

## 静态库

如果你编译的是静态库，你会得到

- libxxx.a

如果你编译的是动态库，你会得到

- libxxx.dll.a
- libxxx.dll

`.dll.a` 是动态库的导出库，在链接过程中会使用到，所以你需要使用 `#pragma
comment(lib, "libxxx.dll.a")` 这种方式在`fore-end`中使用它完成编译，
`libxxx.dll` 在运行中需要使用，你可以把他放到输出的`bin`文件的同一目录或者系统动
态库查找路径的目录中。

# 使用方式

`libxxx.a`这种类型的 C++ 库是没有办法使用的，因为 MSVC 使用静态库格式是
`libxxx.lib`。 `.a` 和 `.lib` 格式的C++库内部结构不一样，直接使用会报错说格式不
对。

`libxxx.dll`这种类型的动态库也没有办法直接使用。MinGW使用GCC编译器，而MSVC 使用
的是微软的编译器 CL。这两种方式对于 C++ 的 `name mangling` 的实现方式不一样。所
以你没有办法直接在 MSVC 上使用 MinGW 编译的库文件，因为在最终链接在一起的时候，
同一个名字会因为不同的`name mangling` 而变成不同的名字，最终导致无法链接成功。

幸运的是，C++兼容C代码，而C的名称是可以不进行`name mangling`的，所以为了能够使得
MSVC能够使用MinGW编译的C++库，你需要给你的库添加一层C接口，并给这些C接口加上：
`extern "C"`修饰使得这些库拥有C链接属性。可参考`stack ovferflow`上的一些答案
[^1][^2][^3][^4]

# 参考

[^1]: http://stackoverflow.com/questions/9928764/mingw-static-lib-used-in-a-msvc-project
[^2]: http://stackoverflow.com/questions/2529770/how-to-use-libraries-compiled-with-mingw-in-msvc
[^3]: http://stackoverflow.com/questions/2472924/linking-to-msvc-dll-from-mingw
[^4]: http://stackoverflow.com/questions/2529770/how-to-use-libraries-compiled-with-mingw-in-msvc/3031167#3031167
