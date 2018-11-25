---
title: C++静态检查工具总结
date: 2018-11-24 22:35:32
tags:
 - C/C++
---

最近在尝试做代码审查，发现很多时候我们把时间花在了低级的规范性错误上面，为了节省这一部分的时间，我尝试了各种静态检查工具，这里做个简单的汇总，方便日后的查看。

<!--more-->

# clang-format

严格来说，它不是静态检查工具，而是代码格式化的工具，类似的工具还有`astyle`，但是相对来说，`clang-format`会好用一些，支持的配置参数也多一些。它的使用请参考[Clang-Format Style Options](https://clang.llvm.org/docs/ClangFormatStyleOptions.html)。

使用统一的代码格式化工具，可以极大的代码格式上面的问题，在多人合作的项目中显得特别的有用。

# cpplint

[cpplint][]是`Google`提供的工具，用于检查我们的代码是否符合[Google C++ Style
Guide][guide]，我们目前的编码规范是基于`Google`的规范，所以这个工具基本上可以直接使用。

[cpplint]: https://github.com/cpplint/cpplint
[guide]: https://google.github.io/styleguide/cppguide.html

## 安装

这个工具是Python写的，所以你可以直接通过pip来安装这个工具的最新版本

```bash
pip3 install cpplint
```

## 使用

这个工具的使用比较简单，直接使用命令：

```bash
cpplint <文件名>
```

比如我们有源文件`hello.cpp`如下：

```cpp
#include <iostream>

int main(int argc, char *argv[])
{
    std::string message = "Hello World";
    std::cout << message << std::endl;
    return 0;
}
```

使用`cpplint hello.cpp`我们会得到如下消息：

> hello.cpp:0:  No copyright message found.  You should have a line: "Copyright [year] <Copyright Owner>"  [legal/copyright] [5]
> hello.cpp:4:  { should almost always be at the end of the previous line  [whitespace/braces] [4]
> Done processing hello.cpp
> Total errors found: 2

### 过滤掉我们不要求的项

如果你并不是严格的直接`Google C++ Style Guide`，cpplint可能会报一些你不想要的错误，这个时候，我们可以通过`filter`参数过滤掉这些错误，比如假如你们不执行关于copyright的规定，你可以加上如下`filter`。

```bash
cpplint --filter=-legal/copyright hello.cpp
```

`legal/copyright`这个字段可以在错误列表中找到，注意前面的`-`号，表示不做这个检查，相反的`+`表示做这个检查。

另外一种方式是在你想要忽略的代码后面加上`// NOLINT`

```cpp
int main(int argc, char* argv[])
{ // NOLINT
}
```

### 集成到 VIM 中

每次都需要执行这个命令会显得非常的繁琐，把这个过程自动化的一个方式是集成到你的开发环境中去，在编辑的时候自动执行这条命令。比如我给我的VIM安装了[ale][]插件，同时配置`cpplint`为检查工具。具体的配置方法，请查看`ale`的官方文档。

[ale]: https://github.com/w0rp/ale

### 在代码 commit 之前自动检查

当然如果你的开发环境不支持这种集成，你可以把这个过程放到`git`的`pre-commit`或者`pre-push`之前，下面是来自`brickgao`的一段[gist][]

[gist]: https://gist.github.com/brickgao/fb359764d46f9c96dd3af885e94b0bab

```bash
#!/bin/sh
#
# Modified from http://qiita.com/janus_wel/items/cfc6914d6b7b8bf185b6
#
# An example hook script to verify what is about to be committed.
# Called by "git commit" with no arguments.  The hook should
# exit with non-zero status after issuing an appropriate message if
# it wants to stop the commit.
#
# To enable this hook, rename this file to "pre-commit".

if git rev-parse --verify HEAD >/dev/null 2>&1
then
    against=HEAD
else
    # Initial commit: diff against an empty tree object
    against=4b825dc642cb6eb9a060e54bf8d69288fbee4904
fi

# Redirect output to stderr.
exec 1>&2

cpplint=cpplint
sum=0
filters='-build/include_order,-build/namespaces,-legal/copyright,-runtime/references'

# for cpp
for file in $(git diff-index --name-status $against -- | grep -E '\.[ch](pp)?$' | awk '{print $2}'); do
    $cpplint --filter=$filters $file
    sum=$(expr ${sum} + $?)
done

if [ ${sum} -eq 0 ]; then
    exit 0
else
    exit 1
fi
```

# cppcheck

cppcheck是一个历史比较悠久的静态检查工具，它的侧重点在于检查代码的逻辑而不是代码的风格。它可以和`cpplint`一起使用，相互补充。

## 安装

绝大部分的系统，可以直接通过包管理器安装`cppcheck`，以`Ubuntu`为例：

```bash
sudo apt install cppcheck
```

## 使用

`cppcheck`比`cpplint`方便一些的地方在于，它支持目录检查，而`cpplint`只支持文件检查。

```bash
cppcheck src
```

表示检查`src`路径下面的所有文件

## 配置

和`cpplint`一样，这个工具支持过滤，你可以通过`--enable=all`打开所有的过滤器

```bash
cppcheck --enable=all src
```

具体支持的filter，可以通过`cppcheck --help`中的`--enable`选项说明来查看，这不再赘述。

## 不足

这个工具最大的问题在于比较容易误判，所以我通常很少会直接使用`--enable=all`，而是直接使用默认的配置。

# clang-check、clang static analyzer、clang-tidy

把这三个工具放一块儿讲是因为它们实际上都是`clang`衍生出来的工具，极其类似互有交叉，大部分情况下我们使用其中之一就够了（我实际上不清楚为什么会有三个这样的工具同时存在）。

这几个工具实际上是编译器级别的检查，它们需要编译文件从而检查代码，所以理论上他们的可靠性会比`cpplint`和`cppcheck`要强一些，同时它的耗时也会它们长一些。

目前我使用的是`clang-tidy`，它和`clang-check`非常类似，但是它支持扩展自定义的检查。另外官方提供`run-clang-tidy`脚本用于实现对整个项目的文件做检查，用起来非常的方便。

## 安装

在`Ubuntu`中使用如下命令可以安装`clang-tidy`

```bash
sudo apt install clang-tidy
```

## 使用

`clang-tidy`的使用，比前面提到的工具要复杂一些，因为它需要编译源码，所以需要知道怎么编译源文件。我们可以通过`CMake`来生成这个编译命名文件：

```bash
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ...
```

`DCMAKE_EXPORT_COMPILE_COMMANDS`这个选项会生成一个叫`compile_commands.json`的文件，有了这个文件，我们可以直接在编译目录下执行`run-clang-tidy`命令，对整个项目做静态的检查。

## 配置

`clang-tidy`支持的检查项特别多，你可以通过：

```bash
clang-tidy -list-checks -checks=*
```

来查看所有支持的检查，使用：

```bash
clang-tidy -list-checks
```

来查看所有已经`enable`的检查，具体的文档请参考[clang-tidy][tidy]

**注意：如果你使用的是zsh，上面的通配符会失效，你可以切换到bash执行这些命令，或者使用`setopt no_nomatch`来关闭zsh对于通配符的拦截**

[tidy]: http://clang.llvm.org/extra/clang-tidy/
