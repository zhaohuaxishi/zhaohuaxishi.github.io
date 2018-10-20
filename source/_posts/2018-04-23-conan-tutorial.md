---
title: C++包管理器——conan
date: 2018-04-23 20:14:31
tags:
    - C/CPP
---

C++可以说是社区驱动型语言，它不像Java和Go背后有主导的公司在推，它的发展更多靠的是由各路专家组成的标准委员会。所以一直以来，它饱受诟病的一点是较难统一，虽然有统一的标准，但是不同的组织有不同的实现和扩充，不同的构建方式，不同的包管理工具。

近年来，`CMake`慢慢的成为了`C++`项目构建方式的事实标准，而这篇文章要介绍的是个人认为很有可能在未来几年成为C++包管理工具事实标准的：`conan`。

<!--more-->

# 官方文档

本文只是一个入门性质的教程，讲解我个人在使用理解`conan`的时候的心得以及遇到的一些问题。推荐大家在看完本文之后找一个时间完成的阅读一下最权威的[官方文档](http://docs.conan.io/en/latest/)。

# 包管理器

很多语言都有自己的包管理器，比如Python的PyPi，C#的Nuget，Rust的cargo。我们可以看一下下面这个直观的例子——gRPC的使用——来感受一下有包管理工具和没有包管理器的区别：

## C++ 版本（Ubuntu为例子，Windows 更复杂）

### Pre-requisites

```shell
$ [sudo] apt-get install build-essential autoconf libtool pkg-config
```

### Protoc

```shell
$ cd grpc/third_party/protobuf
$ sudo make install   # 'make' should have been run by core grpc
```

### Build from Source

```shell
 $ git clone -b $(curl -L https://grpc.io/release) https://github.com/grpc/grpc
 $ cd grpc
 $ git submodule update --init
 $ make
 $ [sudo] make install
```

## Android 版本

```gradle
compile 'io.grpc:grpc-okhttp:1.11.0'
compile 'io.grpc:grpc-protobuf-lite:1.11.0'
compile 'io.grpc:grpc-stub:1.11.0'
```

眼神再不好使，大概也能看出，Android版本会比C++版本简单很多，更重要的是Android版本除了拷贝上面这段代码之外其他的基本都是自动化的过程，而C++版本需要你各种手动的输入和折腾才能侥幸得到你想要的结果。

包管理器有最大的好处在于，绝大部分操作都是自动化的，所以它的操作很简单，基本不会出现错误。作为C++的死忠粉，我除了仇视Android的开发者之外没有其他选择。直到有一个天我遇到`conan`，这个跨平台的C++包版本管理器，我终于可以像下面使用Windows的gRPC：

```toml
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

```shell
$ pip install conan
```

# 使用

conan的使用其实分为两个角色：包的使用者和包的创建者，这一节重点介绍包的使用者的操作，下一节介绍包的创建者的操作。

**本小节以假设你使用CMake来做自动构建，其他的自动构建工具大同小异，我会在后文中给出参考文档**

## conanfile.txt

`conan`的使用比较方便，我们只需要一个配置文件`conanfile.txt`【2】，用于写明我们需要直接依赖的包即可（conan会自动处理依赖的传递）：

```toml
[requires]
zlib/1.2.11@conan/stable

[generators]
cmake
```

## conan install

然后执行下面这条命令：

```shell
$ conan install .
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

大部分的包既提供动态库版本又提供动态库版本，大部分包默认情况下自动安装`static`版本的包，如果你需要使用`shared`版本的包，通常可以在配置文件中加入`shared`这个`option`（注意，并不是所有的包都会提供这个选项）：

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

上面的`-s`表示`setting`，主要包括`build_type`，`compiler`，`arch`等。设置和选项最大的不同在于，设置是`conan`内置的，每个包都存在，而`option`是每个包单独定义的，不同的包可能有不同的`option`（虽然像shared这样的option基本上都会有，理论上更像是setting，conan最新的版本默认都加上这个选项，个人感觉更像是conan的设计失误的一种弥补范式）。

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

好吧，我得承认，上面一小节其实带着忽悠的成分。因为实际上`conan`目前的生态并不是特别完善，所以很多时候，你可能找不到你想要的包，很多时候你可能没有办法直接通过`install`安装你要的包的依赖。

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

虽然它们出现在同一个名字中，但它们不在同一个位置设置。前面两个通常写在配置文件中，而后面两个在命令行中指定。

## 包的创建步骤

创建一个包，实际上就是编译一个包的过程，只不过`conan`把这个过程脚本化来而已。所以在创建一个包之前，我们先整理一下编译一个包需要的步骤：

- 重复下面这三个步骤，直到所有依赖编译完成，后然执行下面三个步骤编译自身
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
    license = "MIT"
    url = "https://github.com/hello/hello.git"
    description = "Hello conan"
    settings = "os", "compiler", "build_type", "arch"
    options = {"shared": [True, False]}
    default_options = "shared=False"
    generators = "cmake"

    def source(self):
        self.run("git clone https://github.com/hello/hello.git")

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

你可以使用下面的命令来创建这个脚本：

```
conan new Hello/0.0.1
```

`conanfile.py`是一个合法的Python脚本，里面定义了一个继承自：ConanFile的类。这个类包括两个部分：属性和方法，属性用于设置一些只读信息而方法用于自动化打包的逻辑。下面我们分步骤讲解一下Conan的打包过程。

### 依赖

conanfile.py提供了两种方式来声明包的依赖，属性`requires`和`requirements`成员函数。通常如果依赖逻辑比较简单，我们可以直接设置属性。

```Python
class MyLibConan(ConanFile):
    requires = (("Hello/0.1@user/testing"),
                ("Say/0.2@dummy/stable", "override"),
                ("Bye/2.1@coder/beta", "private"))
```

依赖本身有两种属性，`override`和`private`。前者出现在需要覆盖依赖的依赖的时候；而后者用于限定内部依赖，比如动态库中依赖的静态库。

如果依赖的逻辑比较复杂，比如需要根据不同的option和setting来设定，我们可以在`requirements()`成员函数中声明依赖：

```Python
def requirements(self):
    if self.options.myoption:
        self.requires("zlib/1.2@drl/testing", private=True)
    else:
        self.requires("opencv/2.2@drl/stable", override=True)
```

这个成员函数最终调用的是`requires()`函数，这个函数同样可以设置`private`和`override`属性。

### 下载源码

依赖处理完之后，我们可以正式的编译我们自己的包来，第一步要做的就是获取源码，同样源码获取其实分为两种情况，一种是使用`exports_sources`属性，一个使用`source()`成员函数。

使用哪一种方式主要看你的`recipe`文件（conanfile.py）是否和源码放在一起。假如你是这个包的开发者，通常你可以把你的recipe和你的源码放一起托管到同一个仓库，假如你只是打包人员【4】，通常你的`recipe`和源码不在同一个仓库。

如果`recipe`和源码在同一个仓库，通常使用`exports_sources`，否则使用`source()`成员函数。

#### exports_sources

这个属性可以用于导出当前仓库下的源码，比如：

```Python
exports_sources = "include*", "src*", "!src/build/*"
```

`recipe`中的大部分属性如果支持多个都是以这种tuple的形式设置，因为它们是只读的。如上所示，我们可以使用通配符`*`也可以排除单独的文件`!`。

#### source() 方法

如果你需要手动下载代码，你可以定义这个成员函数，然后在函数内部编写源码的获取逻辑，最常用的两种方式是：`git clone`和下载源码包。

```Python
def source(self):
    self.run("git clone https://github.com/openssl/openssl.git")
```

```Python
source_tgz = "https://www.openssl.org/source/openssl-%s.tar.gz" % version

def source(self):
    tools.download(self.source_tgz, "openssl.tar.gz")
    tools.unzip("openssl.tar.gz")
```

conan给我们提供了大量的工具辅助我们编写`recipe`，这些工具大多集中在`tools`模块里面，比如我们上面用到的`tools.download`和`tools.unzip`

### 编译

这个过程可能是整个过程中最复杂的操作，因为C++没有统一的构建工具（CMake慢慢的在变成事实标准，但是目前还有大量的历史遗留项目不支持CMake）。幸运的是，conan本身提供了很多封装来帮助我们减少这个过程的复杂性。

#### CMake

如果你在编写一个新项目，很有可能你也在使用CMake（如果不是，推荐你试试），如果你使用CMake，那么构建实际上也非常简单，因为conan提供了CMake这个类帮我们简化编译流程：

```Python
def build(self):
	cmake = CMake(self)
	cmake.configure()
	cmake.build()
```

#### AutoToolsBuildEnvironment

在CMake流行之前，在泛UNIX圈我们通常使用`AutoTool`来自动构建我们的项目，如果你恰好需要处理这种项目，我们可以使用Conan提供的`AutoToolsBuildEnvironment`来编译项目。

```Python
def build(self):
    env_build = AutoToolsBuildEnvironment(self)
    env_build.configure()
    env_build.make()
```

#### MSBuild

如果你的项目需要在Windows下使用，但又没有使用CMake，很有可能你用来VS的MSBuild来做自动构建，同样conan为我们提供了`MSBuild`类来简化编译逻辑：

```Python
def build(self):
	msbuild = MSBuild(self)
    msbuild.build("MyProject.sln")
```

#### 同时兼容多种编译方式

如果你没有办法使用统一的方式处理所有平台的编译，我们可以根据settting或options动态选择：

```Python
def build(self):
    if self.settings.os == "Windows":
        build_with_msbuild(self)
    else:
        build_with_cmake(self)
```

### 打包

编译完成之后，我们需要对编译输出打包，在conan中打包分为两种情况，主要得看我们的自动构建系统是否已经做了打包这个环节。

#### 构建脚本包含打包

如果我们使用CMake或者使用`AutoTool`，我们通常会在编译脚本中指定要安装的文件，这样在执行`make install`的时候，可以把我们想要的头文件和库文件都安装到指定的目录。如果我们的构建脚本中包含这些操作，那么打包这一步我们可以什么都不做，只需要在编译的时候加上`install`这个步骤就可以了。

如果你使用`CMake`你可以调用`install()`函数：

```Python
def build(self):
	...
	cmake.install()
```

如果你使用`AutoTool`，你可以使用`make()`函数并指定`install`参数

```Python
def build(self):
	...
    env_build.make(args=['install'])
```

#### 构建脚本不包含打包

如果你的构建脚本中没有包含打包这个过程，你可以通过conan提供的`package()`成员函数来完成打包，你可以自动拷贝你想要的文件。

```Python
def package(self):
	self.copy("*.h", dst="include")
	self.copy("*.so", dst="lib", keep_path=False)
```

conan会自动查找符合条件的文件，并拷贝到最终的输出目录下面。这个操作对于`MSBuild`的编译比较方便。

### 配置包信息

打包的过程实际上到上面已经结束了，最后这个步骤其实是设置包的信息，以便使用者能够正常的使用，最常见的操作是设置`self.cpp_info.libs`这个属性，它用来告诉使用者在使用的时候需要链接什么库。

```
def package_info(self):
	self.cpp_info.libs = ["hello"]
```

上面这段代码表示使用的时候需要在库引用列表中加上`hello`（具体的设置方式还要看使用者用的是什么编译器）。

## 创建本地的conan包

有了上面脚本，我们可以使用下面的命令来创建一个conan包：

```
conan create . guorongfei/testing
```

它会把`recipe`和相关的文件拷贝到本地的缓存中，然后根据`recipe`创建一个包。本地缓存通常放在`$HOME/.conan/data`目录下，我们可以直接在这个目录下面找到我们刚刚创建的包，通常这个包里面会包含下面几个目录，

- export
- export_source
- source
- build
- package

其中 export 里面存放了我们的`recipe`，剩下的几个目录的功能我们在前面`conanfile.py`的讲解中有讲解，这里不赘述。`package`中存放了我们最终打包出来的文件，如果你想知道自己打的包对不对，可以检查一下这个目录。

### conan 打包的内部过程

conan打包的内部过程可以用下面这张表来描述：

![打包流程](http://docs.conan.io/en/latest/_images/package_create_flow.png)

conan create 把文件拷贝到本地缓存中，之后把文件拷贝到source目录下，执行source()函数下载代码，然后把文件拷贝到build目录下（没一种配置都会有一个相应的build目录），执行build()函数，最后执行package()函数把最终的输出拷贝的对应的`package`目录。

如果我们的编译脚本中包含了`install`这个步骤，所有的文件会被install到package目录下，所以可以不用编写package()再做拷贝。

### 只打包不编译

上面提到，不同的成员函数对应打包的不同的步骤，如果我们没有办法获取源码，只能获取到二进制文件，但是又不想自己去设置库的路径，conan给我们提供来一种方式：只打包不编译。

因为我们不需要编译，所以`recipe`中绝大部分的函数我们都可以不重载，通常我们只需要重载`packet()`和`packet_info()`方法。假如我们已经下载好了所有需要的文件，我们可以这样写：

```Python
from conans import ConanFile, CMake, tools


class HelloConan(ConanFile):
    name = "Hello"
    version = "0.0.1"
    license = "MIT"
    url = "https://github.com/hello/hello.git"
    description = "Hello conan"
    settings = "os", "compiler", "build_type", "arch"
    options = {"shared": [True, False]}
    default_options = "shared=False"
    generators = "cmake"

    def package(self):
        self.copy("*")

    def package_info(self):
        self.cpp_info.libs = ["hello"]
```

当然我们也可以重载`build()`函数从构建服务器上自动下载已经编译好的二进制包进行打包。如果我们只打包不编译，我们可以使用`export-pkg`命令创建包：

```
conan export-pkg . Hello/0.0.1@conan/testing
```

## 把本地的conan包上传到远程的仓库

上面这些步骤让我们在本地创建了一个conan包，但是独乐乐不如众乐乐，我们最后一个步骤通常是把包上传到远程仓库。这里涉及到三个步骤：

### 1. 添加远程仓库

```
conan remote add remote_name remote_url
```

### 2. 获取上传权限

下面这条命令可以用于获取远程仓库的权限，通常下载不需要权限，但是上传需要。

```
conan user
```

### 3. 上传包

因为我们可能创建了多个包（不同到setting，不同到options），我们可以加上 `--all` 表示上传所有到包。

```
conan upload Hello/0.0.1@conan/testing --all
```

`conan`是一个去中心化的包版本管理工具，模型和`git`十分相识。

# 美中不足

我非常希望`conan`是一个完美的包版本管理器，但是它毕竟不是，我个人使用过程中最大的麻烦是他们对于Android的交叉编译支持其实不是特别友好。

## 交叉编译

交叉编译是指在一个架构中编译另一个架构的包，比如在Linux上编译Android的包。conan对于交叉编译的支持是通过创建`toolchain`，并且设置`CXX`、`CC`、`SYSTEM_ROOT`等环境变量来实现的，这种方式在`AutoTool`流行的年代比较流行，目前很多嵌入式开发依旧使用这中方式。具体的使用方式参考官方给的例子：

- [Cross building Boost C++ libraries to Android with Conan](http://blog.conan.io/2018/01/30/Cross-building-Boost-Android.html)

但是Android的NDK，现在也支持使用`toolchain`这种方式，但是这种方式在后续会被慢慢的移除掉。NDK目前的交叉编译使用的是CMake的toolchain.cmake这种方式，直接使用CMake系统来完成交叉编译，它不需要单独制作`toolchian`，使用起来其实方便很多。但是目前conan并不支持这种方式。

# 使用技巧

使用了一段时间的conan之后，积累了一些相关的经验补充在这里：

## 使用alias创建别名包，避免频繁的更新依赖

我们在创建和使用一个conan包的时候都需要指定这个conan包的版本

```
# 创建
conan create . HelloConan/0.1.0@conan/teting

# 使用
conan install HelloConan/0.1.0@conan/teting
```

这给使用上带来的问题是，如果我更新了一个包，但是并没有改变他的接口，使用者依旧
需要更新自己的依赖：比如更新`conanfile.txt`。使用者很多时候只要前后版本可以兼容
，使用者通常自是想要使用最新版本就可以了。

conan提供了`alias`来实现最新版本这个概念，比如如果我们刚刚创建了`0.1`系列最新的包`HelloConan/0.1.5@conan/teting`这个包，我们可以使用下面的命令把最新版本设置为这个包：

```
conan alias HelloConan/0.1@conan/testing HelloConan/0.1.5@conan/testing
```

## 手动创建包，避免每次更新都重新编译整包

`conan create`这条命令的背后实际上执行了整个打包流程【5】，这条命令很方便，但是
会导致所有的包都重新编译一遍，而很多时候其实我们只是做了非常小的一个改动而已。
为了充分的利用编译缓存，我们可以手动的执行打包流程，也就是手动执行`conan
create`背后的指令：

```
conan install . --install-folder=./build
conan build . --source-folder=./ --build-folder=./build
conan export-pkg . HelloConan/0.1.0@conan/testing --package-folder=./build/package
```

`conan create`实际上依次执行了`conan source`，`conan install`，`conan build`，`conan package`，`conan export-pkg`这几条命名。它之所以慢是因为每次都需要重新执行这些命令，如果我们手动创建包，我们可以有下面这些改进：

- 不执行`source`，因为源码就在我们手上，可能就是当前这个目录
- 选择性执行`conan install`，依赖项实际上我们可以只安装一次，后续的编译不需要重复执行
- 指定同一个编译目录，这个是最见效的方式，我们可以把编译目录手动设置为固定的目录，这样可以充分的利用编译缓存来极大是缩短编译时间。
- 不执行`conan package`，因为`conan export-pkg`会执行`package()`函数，也就是`conan package`命令做的事情。

**需要注意的是，`export-pkg`包含两种不同的执行模式，如果我们在`build`这个步骤中使用了`cmake.install()`创建了包，我们只需要指定`package-folder`就可以了（这种情况通常我们不写`package`函数），否则我们需要指定`source-folder`和`build-folder`，以便执行`package`函数打包。**


---

【1】：为什么使用脚本语言来实现编译语言的包版本管理器可以参考官方的解释：[Why a C++ package manager can't be written in C++](http://blog.conan.io/2016/09/27/Why-a-C++-package-manager-can't-be-written-in-C++.html)

【2】：这个文件的格式比较像`toml`，但我没有找到官方的说法。

【3】：Windows 中的 HOME 目录通常是用户目录的根目录。

【4】：比如目前网络上存在大量的开源C/C++项目没有conan包，我们可以为这些包编写打包脚本

【5】：https://docs.conan.io/en/latest/developing_packages/package_dev_flow.html
