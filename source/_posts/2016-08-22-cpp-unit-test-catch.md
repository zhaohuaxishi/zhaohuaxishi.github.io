---
title: C++ 的单元测试工具 —— Catch
date: 2016-08-22 15:37:30
tags:
 - C/CPP
---

如果你平常使用 Java 语言做开发，当你听到单元测试工具的时候，你很可能马上会想起
JUnit。作为一名C++软件工程师，当我第一次打算给我的程序做单元测试的时候，我的第一
想法是：有这样的工具吗？经过一段时间的搜索之后，我的反应变成了：我该用哪一个？

我在学校的时候，很少听说C++的单元测试工具，以至于我一直认为这样的工具是不存在。
后来慢慢的发现我们可以选择的远比你想象中的要多得多：Catch, Boost.Test,
UnitTest++, lest, bandit, igloo, xUnit++, CppTest, CppUnit, CxxTest, cpputest,
googletest, cute。

那我们应该使用哪一个呢？如果你在 Google 里面搜索：`best c++ unit testing
framework`。头两条，一条是 [stackoverflow][] 的问答，另一条是 [reddit][] 的问答
。这两个问题都指向同一个单元测试框架：[Catch][catch]。

[stackoverflow]: http://stackoverflow.com/questions/20606793/state-of-the-art-c-unit-testing
[reddit]: https://www.reddit.com/r/cpp/comments/36pru0/best_way_to_do_unit_testing_in_c/
[catch]: https://github.com/philsquared/Catch

<!--more-->

# 为什么使用 Catch

在 Catch 的官方文档中有一篇：[Why do we need yet another C++ test
framework?][whycatch] 有兴趣的可以去看看。对我来说，它最吸引我的地方主要是：

[whycatch]: https://github.com/philsquared/Catch/blob/master/docs/why-catch.md

- 几乎不用配置，它是一个单头文件的测试框架，压根不要什么额外的配置就可以使用
- 语法非常简单明了，用它写的测试代码和自然语言一样易懂。

# 如何使用它

`Catch` 是单头文件库，你直接 #include "catch.hpp" 它就可以了。然后你就可以像下面
这样写测试代码：

```
SCENARIO( "vectors can be sized and resized", "[vector]" ) {

    GIVEN( "A vector with some items" ) {
        std::vector<int> v( 5 );

        REQUIRE( v.size() == 5 );
        REQUIRE( v.capacity() >= 5 );

        WHEN( "the size is increased" ) {
            v.resize( 10 );

            THEN( "the size and capacity change" ) {
                REQUIRE( v.size() == 10 );
                REQUIRE( v.capacity() >= 10 );
            }
        }
        WHEN( "the size is reduced" ) {
            v.resize( 0 );

            THEN( "the size changes but not capacity" ) {
                REQUIRE( v.size() == 0 );
                REQUIRE( v.capacity() >= 5 );
            }
        }
        WHEN( "more capacity is reserved" ) {
            v.reserve( 10 );

            THEN( "the capacity changes but not the size" ) {
                REQUIRE( v.size() == 5 );
                REQUIRE( v.capacity() >= 10 );
            }
        }
        WHEN( "less capacity is reserved" ) {
            v.reserve( 0 );

            THEN( "neither size nor capacity are changed" ) {
                REQUIRE( v.size() == 5 );
                REQUIRE( v.capacity() >= 5 );
            }
        }
    }
}
```

这几乎是不需要解释就可以理解的读懂的代码。这种测试方式称为 BDD（Behaviour Driven
Development），是最新的一种测试方式，它强调的是“行为”而不是“测试”，有兴趣可以看
看[这篇文章][bdd]。

[bdd]: https://dannorth.net/introducing-bdd/

如果你习惯传统的TDD测试，你可以像下面这样写测试代码：

```
TEST_CASE( "vectors can be sized and resized", "[vector]" ) {

    std::vector<int> v( 5 );

    REQUIRE( v.size() == 5 );
    REQUIRE( v.capacity() >= 5 );

    SECTION( "resizing bigger changes size and capacity" ) {
        v.resize( 10 );

        REQUIRE( v.size() == 10 );
        REQUIRE( v.capacity() >= 10 );
    }
    SECTION( "resizing smaller changes size but not capacity" ) {
        v.resize( 0 );

        REQUIRE( v.size() == 0 );
        REQUIRE( v.capacity() >= 5 );
    }
    SECTION( "reserving bigger changes capacity but not size" ) {
        v.reserve( 10 );

        REQUIRE( v.size() == 5 );
        REQUIRE( v.capacity() >= 10 );
    }
    SECTION( "reserving smaller does not change size or capacity" ) {
        v.reserve( 0 );

        REQUIRE( v.size() == 5 );
        REQUIRE( v.capacity() >= 5 );
    }
}
```

实际上这两种方式是等价，SCENARIO 只是 `TEST_CASE` 的别名，GIVEN、WHEN、THEN 最终
也是 map 到 SECTION 上面的。这其中的差异只是存在于测试的思维不同而已，你完全可以
根据自己的喜好使用你最喜欢的方式即可。

# SECTION 的执行顺序

上面的代码很清晰易懂，不过有一个地方需要注意，那就是 SECTION 的执行方式。在上一
小节的代码中，`TEST_CASE` 中有 4 个 SECTION，它们并不是单纯的顺序执行关系。在第
一个 SECTION 执行完成之后，会重头开始执行并跳过已经执行过的 SECTION。也就是说上
面的代码的执行路径大概是这样的（去掉了 SECTION 宏之后）：

```
// SECTION 1
std::vector<int> v( 5 );

REQUIRE( v.size() == 5 );
REQUIRE( v.capacity() >= 5 );

v.resize( 10 );

REQUIRE( v.size() == 10 );
REQUIRE( v.capacity() >= 10 );

// SECTION 2
std::vector<int> v( 5 );

REQUIRE( v.size() == 5 );
REQUIRE( v.capacity() >= 5 );

v.resize( 0 );

REQUIRE( v.size() == 0 );
REQUIRE( v.capacity() >= 5 );

// SECTION 3
std::vector<int> v( 5 );

REQUIRE( v.size() == 5 );
REQUIRE( v.capacity() >= 5 );

v.reserve( 10 );

REQUIRE( v.size() == 5 );
REQUIRE( v.capacity() >= 10 );

// SECTION 4
std::vector<int> v( 5 );

REQUIRE( v.size() == 5 );
REQUIRE( v.capacity() >= 5 );

v.reserve( 0 );

REQUIRE( v.size() == 5 );
REQUIRE( v.capacity() >= 5 );
```

# 测试代码的执行入口

在C++中任何代码需要执行，都需要通过 main 函数这个入口，测试代码也不例外。Catch
不需要我们自己编写 main 函数去调用这些测试代码。它提供了默认的 main 函数入口，你
只需要在（而且仅在）一个文件中加入下面的配置宏：

```
#define CATCH_CONFIG_MAIN
#include "catch.hpp"
```

## 最佳实践

最佳实践是单独用一个文件放这两行代码，把测试代码写在其他的文件中。

之所以这样做是因为Catch是单头文件库，这意味着它里面的内容会最终出现在所有的包含
这个头文件的编译单元中。如果我们把测试代码和上面两行代码放在一起会导致每次编译测
试代码的时候都需要编译 Catch 的内核，这会导致编译速度非常非常的慢。如果把两者分
开，Catch 的内核只需要在一个文件中编译一次（因为 Catch 内部做了判断，如果内核编
译过了是不需要再次编译的，即使你在多个文件中使用了 #include "catch.hpp"）。这个
文件的编译速度相对较慢，但是这个文件不会改动所以整个开发周期中它只需要编译一次，
而不断更新的测试代码的编译速度会因此快很多。

## 命令行参数

Catch 提供的这个 main 函数实现的另一个强大的功能是丰富的命令行参数，你可以选择执
行其中的某些 `TEST_CASE`，也可以选择不执行其中的某些 `TEST_CASE`，你可以用它调整
输出到 xml 文件，也可以用它从文件中读取需要测试的用例。这些命令的具体使用请参考
Catch 的官方文档[Command line][cmd]一节。

[cmd]: https://github.com/philsquared/Catch/blob/master/docs/command-line.md

## TAG

需要注意的是，这些强大的命令行大多数是基于 TAG 的，也就是 `TEST_CASE` 定义中的第
二个参数。

```
TEST_CASE( "vectors can be sized and resized", "[vector]" )
```

上面的定义中 "[vector]" 就是一个 TAG，你可以提供多个 TAG：

```
TEST_CASE( "D", "[widget][gadget]" ) { /* ... */ }
```

这样的话你可以在命令行中根据 TAG 去选择是否需要执行该 `TEST_CASE`。比如：

```
./catch "[vector]" // 只执行那些标记为 vector 的测试用例
```

此外你还可以使用一些特殊的字符，比如 `[.]` 表示隐藏。`[.integration]` 则表示默认
隐藏，但是可以在命令行中使用 `[.integration]` 这个 TAG 执行。其他的一些特殊的字符
请参考官方文档的[Test cases and sections][testcase]一节

```
./catch // 默认不执行 integration

./catch "[.integration]" // 使用 TAG 执行 integration
```

[testcase]: https://github.com/philsquared/Catch/blob/master/docs/test-cases-and-sections.md

## 提供自己的 main 函数入口

如果你不喜欢上面的处理方式，想要自己提供 main 函数，你可以使用
`CATCH_CONFIG_RUNNER`，具体的细节请查看官方文档中的 [Supplying main() yourself]
[main]一节。

[main]: https://github.com/philsquared/Catch/blob/master/docs/own-main.md

# 其他内容

其实 Catch 本身相对来说比较简单，不需要太多其他的学习，大部分的用法是非常的直观
的，看完它的[官方教程][tutorial]之后基本上可以上手了，然后有时间慢慢的读一读它的[官方文
档集合][docset]

[tutorial]: https://github.com/philsquared/Catch/blob/master/docs/tutorial.md
[docset]: https://github.com/philsquared/Catch/blob/master/docs/Readme.md
