---
title: C++包管理器——conan
date: 2018-04-23 20:14:31
tags:
    - C/CPP
---

C++可以说是社区驱动型语言，它不像Java和Go背后有主导的公司在推，它的发展更多靠的是由各路专家组成的标准委员会在主导。所以一直以来，它饱受诟病的一点是较难统一，虽然有统一的标准，但是不同的组织有不同的实现和扩充，不同的构建方式，不同的包管理工具。

近年来，CMake慢慢的称为了C++项目构建方式的事实标准，而这篇文章要介绍的是个人认为很有可能在未来几年成为C++包管理工具事实标准的：conan。

<!--more-->

# 官方文档

本文只是一个入门性质的教程，讲解我个人在使用理解conan的时候遇到的一些问题。推荐大家在看完本文之后找一个时间完成的阅读一下最权威的[官方文档](http://docs.conan.io/en/latest/)。

# 包管理器

大部分语言有自己的包管理器，比如Python的PyPi，Java的Maven，C#的Nuget，Rust的cargo。它们最大的好处就是方便模块的复用，在使用包管理器之前，我们想用别人的库，通常的步骤是：

0. 对于所有的依赖项目和项目本身循环执行下面这几个步骤
1. 下载源码
2. 编译
3. 把生成的库文件和头文件拷贝到自己的工程目录
4. 更新头文件引用路径和库文件引用路径
5. 编译我们自己的程序

而使用了包管理器之后，我们通常只需要下面几个步骤：

0. 在配置文件中写下你想要复用的模块
2. 安装你想要的模块
3. 编译我们自己的程序

一般来说，如果有IDE支持，第二步和第三步可能是放在一起的。我们可以看一下下面这个直观的例子：gRPC的使用：

## C++ 版本（Ubuntu为例子，Windows 更复杂）

### Pre-requisites

```shell
$ [sudo] apt-get install build-essential autoconf libtool pkg-config
```

### Protoc

```
$ cd grpc/third_party/protobuf
$ sudo make install   # 'make' should have been run by core grpc
```

### Build from Source

```
 $ git clone -b $(curl -L https://grpc.io/release) https://github.com/grpc/grpc
 $ cd grpc
 $ git submodule update --init
 $ make
 $ [sudo] make install
```

## Android 版本

```
compile 'io.grpc:grpc-okhttp:1.11.0'
compile 'io.grpc:grpc-protobuf-lite:1.11.0'
compile 'io.grpc:grpc-stub:1.11.0'
```

眼神再不好使，大概也能看出，Android版本会比C++版本简单很多，更重要的是Android版本处理拷贝上面这段代码之外其他的基本都是自动化的过程，而C++版本需要你各种手动的输入和折腾才能侥幸得到你想要的结果。

包管理器有最大的好处在于，绝大部分操作都是自动化的，所以它的操作很简单，基本不错出现错误。作为C++的死忠粉，我除了仇视Android的开发者之外没有其他选择，直到有一个天我遇到`conan`，这个跨平台的C++包版本管理器，我终于可以像下面使用Windows的gRPC：

```
[requires]
gRPC/1.9.1@inexorgame/stable
```

嗯，世界真美好。

# 跨平台的包管理器

实际上，`conan`并不是第一个流行起来的C++包管理器，在VS的生态中，Nuget可以用于管理VS平台的C++包，所以如果你只需要支持Windows平台，你可以直接使用Nuget，因为它和VS的集成度会比较高，在开发上可能便利性可能会超过`conan`。

`conan`最大的优势在于它的跨平台，它可以支持：

- 不同的操作系统（Windows，Linux，macOS，FreeBSD等等）
- 不同的编译器（gcc，msvc，clang等等）
- 不同的构建工具（CMake，QMake，MSBuild，Autotools等等）
- 不同的构建方式（原生编译，交叉编译等等）

如果你需要一个跨平台的解决方案，`conan`可能是目前唯一的选择

# 安装

有意思的是，作为C++的包版本管理器，conan不是用C++来实现的，它甚至不是使用编译型语言来实现的，它使用的是脚本语言Python【1】。所以安装`conan`之前，我们需要先安装Python和pip，然后执行下面的命令安装`conan`包：

```Shell
pip install conan
```

# 使用

conan的使用其实分为两个角色：包的使用者和包的创建者，这一节重点介绍包的使用者的操作，下一节介绍包的创建者的操作。

**本小节以假设你使用CMake来做自动构建，其他的自动构建工具大同小异，我会在后文中给出参考文档**

## conanfile.txt

`conan`的使用比较方便，我们只需要一个配置文件`conanfile.txt`【2】，写明我们需要直接依赖的包即可（conan会自动处理依赖的传递）：

```
[requires]
zlib/1.2.11@conan/stable

[generators]
cmake
```

## conan install

然后执行下面这条命令：

```shell
conan install .
```

上面这条命令中的`.`表示`conanfile.txt`的路径，如果你不是在同一个路径下面（比如在编译路径下），你需要指定相对路径或者全路径。通常上面这条命令会自动安装我们想要的包，然后在在执行`install`命令的路径下生成三个文件：

- conanbuildinfo.txt
- conanbuildinfo.cmake
- conaninfo.txt

其中`conaninfo.txt`这个文件可以用来判断这个包的详细信息，包括编译器信息，系统架构（`x86`，`x86_64`等），通常如果你自动安装出现编译错误时可以考虑查看这个文件来确认一下包的信息和你期望的信息是否一致（比如你想要一个Debug包，但是下载成来Release包）。

`conanbuildinfo.cmake`这个文件是给CMake用的，让它知道如何引用依赖包，比如头文件的引用路径，库的引用路径，库的链接等信息。相对的`conanbuildinfo.txt`这个文件可供我们阅读，排查上面这些信息是否有误。不同的`generators`导致不同的供构建系统使用的文件生成（比如如果指定`generators`是`visual_studio`，会生成`conanbuildinfo.props`），但是统统都会生成供我们阅读的`conanbuildinfo.txt`。

## 引用生成自动生成的编译文件

我们需要额外的设置，把这个生成的文件集成到我们自己的编译系统中去，比如如果我们使用的是`CMake`，我们需要修改我们顶层的的`CMakefile.txt`，加入下面这两句（第一句的`include`怎么写，依赖于我们在哪个路径下执行`conan install`命令）：

```CMake
include(${CMAKE_BINARY_DIR}/conanbuildinfo.cmake)
conan_basic_setup()
```

然后让我们的`target`链接我们的依赖库，

```CMake
target_link_libraries(target ${CONAN_LIBS})
```

上面这个过程通常只需要执行一次，所以其实工作量比我们想象中的要小一些。

## 使用动态库

大部分的包既提供动态库版本又提供动态库版本，默认情况下`conan`自动安装`static`版本的包，如果你需要使用`shared`版本的包，通常可以在配置文件中加入`shared`这个`option`（注意，并不是所有的包都会提供这个选项）：

```
[options]
zlib:shared=True
```

在Windows中，动态库分成两个部分`xxx.dll`会导出库`xxx.lib`，`dll`不用参与链接，但是需要放到可执行目录下，所以使用动态库通常还意味着需要拷贝`dll`到`bin`目录下，`conan`使用`import`这个配置来自动完成这个操作：

```
[imports]
bin, *.dll -> ./bin
lib, *.dylib* -> ./bin
```

## 使用 Debug 版本的库

默认情况下，conan自动安装`Release`版本的包，但是使用`Debug`版本的库对于调试开发其实比较有帮助。和前面不同的是，如果我们想要使用`Debug`版本包，我们通常不是使用`conanfile.txt`而是使用命令行参数：

```
conan install . -s build_type=Debug
```

上面的`-s`表示`setting`，主要包括`build_type`，`compiler`，`arch`等。设置和选项最大的不同在于，设置是conan内置的，每个包都存在，而`option`是每个包单独定义的，不同的包可能有不同的`option`（虽然像shared这样的option基本上都会有，理论上更像是setting，conan最新的版本默认都加上这个选项，个人感觉更像是设计失误的一种弥补范式）。

除了`-s`参数之外，`conan`还提供`-o`参数用于指定选项：

```
conan install . -o zlib:shared=True
```

你甚至可以不提供`conanfile.txt`文件，直接使用命令行完成包的安装：

```
conan install zlib/1.2.11@conan/stable -g cmake -s build_type=Debug -o zlib:shared=True
```

## profile

其实和`option`一样，`setting`既可以用命令行参数指定，也可以通过配置文件指定，只不过`setting`是写在`profile`而不是`conanfile.txt`中。conan安装完之后会有一个默认的profile：`$HOME/.conan/profiles/default`【3】。如果你想要系统默认都下载`Debug`包，你可以修改这个文件，把`build_type`改成`Debug`。

如果你不想影响全局，又不想频繁的输入命令行参数，你可以在`$HOME/.conan/profiles/`下新建一个profile，比如`myproject`，然后使用下面命令安装依赖包：

```shell
conan install . --profile=myproject
```

这种方式在对于简单的参数来说没有太大的意义，但是对于交叉编译的中特别有用，可以避免大量的参数的输入工作。

## 其他工具的集成

前面的例子以`CMake`为例，其他工具的集成使用可以参考下面这个[官方文档](http://docs.conan.io/en/latest/integrations.html)

# 创建一个包

好吧，我得承认，我在上面一小节其实带着忽悠的成分。因为实际上`conan`目前的生态并不是特别完美，所以很多时候，你可能找不到你想要的包，所以你没有办法直接通过`install`安装你要的包的依赖。

俗话说，求人不如求己，我们完全可以自己的打包，自己用。这一些节主要介绍如何创建一个`conan`包。

## 理解conan的包名

在实际创建一个包之前，我们先来理解一下conan包的包名。

```
zlib/1.2.11@conan/stable
```

上面这个包名包含四个部分：

- zlib      名字
- 1.2.11    版本
- conan     用户
- stable    通道（channel）

虽然它们出现在同一个名字中，但它们不在同一个位置指定。前面两个通常写在配置文件中，而后面两个在命令行中指定。

## 包的创建步骤

创建一个包，实际上就是编译一个包的过程，只不过`conan`把这个过程脚本化来而已。所以在创建一个包之前，我们先整理一下编译一个包需要的步骤：

- 重复下面这三个步骤，直到所有依赖编译完成
- 下载源码
- 编译
- 安装

所以创建一个`conan`包大概也就是把这几个步骤自动化而已，自动化这些操作的脚本叫做：`conanfile.py`，官方称之为`recipe`。`recipe`是菜谱的意思，这个词非常好的体现了`conanfile.py`的功能，引导`conan`一步一步的创建一个包。

## conanfile.py

一个 `conanfile.py` 通常的像下面这个样子：

```Python
from conans import ConanFile, CMake, tools


class HelloConan(ConanFile):
    name = "Hello"
    version = "0.0.1"
    license = "<Put the package license here>"
    url = "<Package recipe repository url here, for issues about the package>"
    description = "<Description of Hello here>"
    settings = "os", "compiler", "build_type", "arch"
    options = {"shared": [True, False]}
    default_options = "shared=False"
    generators = "cmake"

    def source(self):
        self.run("git clone https://github.com/memsharded/hello.git")

    def build(self):
        cmake = CMake(self)
        cmake.configure(source_folder="hello")
        cmake.build()

    def package(self):
        self.copy("*.h", dst="include", src="hello")
        self.copy("*hello.lib", dst="lib", keep_path=False)
        self.copy("*.dll", dst="bin", keep_path=False)
        self.copy("*.so", dst="lib", keep_path=False)
        self.copy("*.dylib", dst="lib", keep_path=False)
        self.copy("*.a", dst="lib", keep_path=False)

    def package_info(self):
        self.cpp_info.libs = ["hello"]

```

---

【1】：为什么使用脚本语言来实现编译语言的包版本管理器可以参考官方的解释：[Why a C++ package manager can't be written in C++](http://blog.conan.io/2016/09/27/Why-a-C++-package-manager-can't-be-written-in-C++.html)

【2】：这个文件的格式比较像`toml`，但我没有找到官方的说法。

【3】：Windows 中的 HOME 目录通常是用户目录的根目录。
