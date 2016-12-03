---
title: 使用 Doxygen 生成文档注释
date: 2016-11-20 19:51:32
tags:
toc: true
---

全文是个人关于注释的观点，很可能有误，不想误人子弟，请慎重考虑其中的任何一个观点
，欢迎拍砖。


# 注释不是越多越好

刚开始学编程的时候，总是会有一些老鸟程序员告诉我，优秀的程序员都习惯给自己的代码
加上大量的注释，这一点甚至有很多的老师都是这么教的，以至于有很长一段时间我都对这
个观点深信不疑。

第一个让我这个观点产生怀疑的人是“林纳斯·托瓦兹”，有一段时间疯狂的迷恋操作系统的
内核，而我尝试读一些源码的时候发现，大神们写的代码其实并不会有太多的注释，但是他
们的代码依旧很漂亮。

第二个让我对于注释越多越好的观点产生怀疑的人是C++之父，忘记在他的那一本书中看到
，注释如果和源码不同步比没有注释更糟糕。

注释是你的源码的一部分，只要你选择写下一行注释，你就需要担负起维护它的责任。我们
经常会在重构一段源码的时候忘记给更改的地方的注释加上相应的修改。最后留给源码的阅
读者无尽的苦恼这段代码怎么就能实现注释里面说的这个功能呢。

<!--more-->

# 代码的可读性远远比注释更重要

所有的程序员都应该像白居易学习，把你的代码写得简单明了，逻辑清晰。写一段清晰易懂
的代码，远比写一段晦涩难懂的代码加上一段晦涩难懂的注释要好的多。所以在你打算写下
一段注释之前先看看你的代码，命名是否规范啊？逻辑是否清晰呢？嵌套层次是否合理呢？
函数是否够精简呢？

如果以上的问题你的答案都是“是”，而你依然觉得代码相对难懂，那就加上一些简洁的注释
吧，同时记得在更新代码的时候，更新你的注释。

# 接口的注释

**本文提到的接口指的函数签名，包括普通函数、成员函数等**

以上这些论述只是针对了源码的注释，个人认为接口的注释还是需要尽可能详细的，因为接
口本身通常都是比较稳定的，此外接口本身能够提供的信息也相对较少，而接口作为模块之
间交互的协议规范，不应该有任何模棱两可的东西，需要详细、准确。

# 接口的文档中应该包括哪些内容

个人认为以下信息是需要出现接口的文档中的：

- 接口的功能。
- 接口的前置条件，通常是指输入的参数需要遵循的条件。
- 接口的后置条件，通常是指接口的返回值，或者说接口的`side effect`。
- 接口的注意事项，如果有的话，应该说清楚调用该接口需要特别注意的地方。比如会抛些
  什么异常。

# 使用 Doxygen 生成 C++ 代码文档

Doxygen可以用来从C++的源码注释中抽取文档，目前可以说是C/C++的文档注释的实际标准
。它的使用非常简单，只需要你按照特定格式编写注释，便可以通过一条命令生成各种格式
的文档（html, rtf, latex, xml, man等）。所以关于它的使用我们需要知道两件事情：第
一，怎么编写注释；第二，怎么生成文档。

## 怎么编写注释

嗯，我不是怀疑你不会使用`//`或者`/**/`，我们这里关心的是编写`Doxygen`能够识别的
注释。

### 注释块

你可以有多种方式编写一个`Doxygen`注释块，我只使用其中的 JavaDoc 风格的注释块儿：

```c++
/**
 * comment text here
 */
```


### 标记（命令）

在一个注释块中，你除了可以写一些文字之外，还可以使用一些特殊的标记，比如最常用的
`param`标记。这种特殊标记，官方称为`Special Commands`，你可以在[这里][commands]
中查到相关的信息，如果你英文不够好，可以参考这个简单的[翻译][cmdtranslate]，只可
惜这篇翻译烂尾，只有一部分翻译。网上另有有一份《Doxygen 中文手册》可以参考。

[commands]: http://www.stack.nl/~dimitri/doxygen/manual/commands.html
[cmdtranslate]: http://www.cnblogs.com/benhuan/p/3302114.html

一个标记，你可以有两种书写方式 `\param` 或者 `@param` 个人比较喜欢使用后一种，因
为它和 JavaDoc 的方式一样，比较统一。

```c
/**
 * @param url the media location to play
 */
bool Play(const std::string& url)
```

`Doxygen`给我们提供了大量的特殊标记，让我们可以给注释加上语法本身无法传达的一些
信息。 个人认为这些标记最有效的学习方式是看别人是怎么使用的，目前为止我见过最好
的注释文档是`FFMPEG`的[注释文档][ffmpeg]，这也是我平常学习的时候参考的最多的一份
文档。它的在线版Doxygen文档在墙内访问比较慢，建议下载源码自行生成（在源码的根目
录下执行 `doxygen doc/Doxyfile`）。下面总结了一些个人认为比较重要的标记。

[ffmpeg]: http://ffmpeg.org/

#### @brief

标注用来作为简单的功能描述，这个标记其实可以不写。比如：

```c
/**
 * @brief Play the media at url.
 */
bool Play(const std::string& url)
```

可以直接写成

```c
/**
 * Play the media at url.
 */
bool Play(const std::string& url)
```

#### @param

标注参数，比如上文中的例子： 这个命令可以有[in]、[out] 两个属性，表示该参数是输
入参数还是输出参数。

```c
/**
 * @param[out] dest The memory area to copy to.
 * @param[in]  src  The memory area to copy from.
 * @param[in]  n    The number of bytes to copy
 */
void memcpy(void *dest, const void *src, size_t n);
```

#### @return

标注返回值。返回值是一个接口非常重要的内容，很多人会忽略它的文档说明。

```c
/**
 * @return true on success, false otherwise
 */
bool Play(const std::string& url)
```

#### @throw @throws @exception

异常属于接口的一部分，Java 语言通过异常说明符把这一点内置到了语言当中，C++曾经也
有异常说明符，但是后来被废弃（因为没有办法高效的实现，可参考《C++编程规范》一书
的Item75）。无论语言本身是否支持，异常都应该是接口的一部分，没有人愿意调用一个不
知道会抛什么异常的函数。

```c
/**
 * @throw std::out_of_range if idx >= vector.size()
 */
bool vector::at(size_t idx)
```

#### @warning

接口容易出错的地方。

```c
/**
 * @warning application may crash if dest or src is null
 */
void memcpy(void *dest, const void *src, size_t n);
```

使用这个标记标注的文字会以一个红色的标记出现最终的文档中，起到非常好的警告作用。

#### @note

标注需要特别注意的地方，可以用来标注接口的前置条件：

```c
/**
 * @note do nothing if url is invalidate
 */
bool Play(const std::string& url)
```

#### @deprecated

这个标记对于C/C++来说是非常有用的，因为语言本身没有提供这样的特性，所以文档中的
说明就显得非常重要了。

```c
/**
 * @deprecated this function is deprecated, since it has the same effect of
   callling SeekTo(from) after Play(url).
 */
bool Play(const std::string& url, size_t from)
```

#### @see @sa

相当于 JavaDoc 的 see also 功能。

#### @since

标记接口的版本号，Android 中有大量的接口是在不同的版本中逐渐添加进去的，对于库来
说这个比较比较重要。

### 工具推荐

这些标记如果要纯手写的话胡非常麻烦，通常你都可以在你喜欢的编辑器或者IDE中找到相
应的插件。比如Vim的插件`vim-scripts/DoxygenToolkit.vim`，用的工具不重要，重要的
使用同一种编写风格。

## 如何生成文档

使用特定的格式编写完注释之后，你可以使用`doxygen`命令生成特定格式的文档（前提是
你已经安装了Doxygen，它是开源跨平台的，所以你急本上可以在任何一个平台使用它）。
生成文档分成三个步骤：

1. 生成配置文件
2. 修改配置文件
3. 生成文档

### 生成配置文件

这个步骤很简单：

```bash
doxygen -g <config-file>
```

如果你不提供文件名字（config-file），你会得到一个叫做`Doxyfile`的配置文件。通过
修改配置文件，你可以控制`doxygen`的最终输出接口。

所有的选项都是以键值对的形式配置的，你直接使用你最喜欢的文件编辑器修改
`Doxygenfile`即可，当然你可以使用图形界面工具`Doxywizard`来完成编辑。Doxygen提供
了非常多的可配置选项，通常大部分的值你直接使用默认值就可以了，下面这里列出了其中
比较重要的一些配置选项以及它们的含义。

### 修改配置文件

`Doxyfile`中的的配置选项非常的多，大部分情况下使用默认值就可以了，每一个配置选项
中之前会有一段注释，有时间可以通读一遍了解各个选项的作用。这些选项中有一些需要特
别注意的在[学习用doxygen生成源码文档][ibm]一文中已经提到了这里就不再重复了，这里
补充几个没有提到的。

#### `EXAMPLE_PATH`

例子代码的位置，你可以为你的API提供一些demo例子，指定这个配置会让你的接口描述下
面多出一栏 Example，Doxygen会自动查找这个接口在demo中出现的位置，给出链接，方便
用户学习[^1]。

#### `REFERENCED_BY_RELATION` `REFERENCES_LINK_SOURCE`

如果你想要Doxygen生成的文档能够方便的跳转，把这两个选项设置为`YES`.

如果不想自己动手配置这些选项，你可以考虑下载一份FFMPEG的配置选项，在它的基础上去
掉你不需要的功能就可以了。

####

配置文档的输出位置


### 生成文档

修改完成配置文件之后，你就可以使用下面的命令生成文档：

```
doxygen /path/to/Doxyfile
```

如果你配置了html输出，你最终会得到一个`html`文件夹（在todo配置的文件夹下面），打
开里面的 index.html 就可以开始浏览文档了。

# 参考

1. [Doxygen Document](http://www.stack.nl/~dimitri/doxygen/manual/index.html)
1. [学习用 doxygen 生成源码文档][ibm]

[ibm]: http://www.ibm.com/developerworks/cn/aix/library/au-learningdoxygen/index.html
[^1]: http://ffmpeg.org/doxygen/3.2/group__lavf__core.html#gac7a91abf2f59648d995894711f070f62
