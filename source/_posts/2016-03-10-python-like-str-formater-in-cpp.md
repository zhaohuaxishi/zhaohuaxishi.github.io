title: C++中实现简单的Python风格字符串格式化函数
date: 2016-03-10 16:44:04
tags:
 - C/CPP
---

前一段时间学习 `Python`，被这门语言的便利性惊艳到了，比如你可以这样产生一个字
符串：

```python
# output "hello world, PI is 3.14"
"{} world, PI is {}".format("hello", 3.14)
```

最近学习C++标准库，看到C++11新的变参模板，发现它可以用来实现一个简单的类似
`Python` 风格字符串格式化函数[^1]，语法如下：。

```cpp
// output "hello world, PI is 3.14"
FormateString("{} world, PI is {}", "hello", 3.14)
```

本文讲解一个非常简单的实现版本，不处理下面这样的语法：

```cpp
// output "say you say me"
FormateString("{0} you {0} me", "say")
```

如果你需要一个丰富的字符串格式化功能，可以考虑使用[cppformat][]这个库。

[cppformat]: https://github.com/cppformat/cppformat

<!--more-->

# 具体实现

**注意：因为使用了C++11中的新特性，你需要一个C++11编译器，通常你需要在编译的时
候设置 `-std=c++11`。本文实现的版本源码可以在[我的代码库][repo]中找到**

[repo]: https://github.com/zhaohuaxishi/code-snippet/

基本的实现思路是使用变参模板捕获格式说明符之外的所有参数，然后依次把他们通过
`stringstream` 拼接在一起，最终返回拼接的结果，函数的声明如下：。

```cpp
template <typename... Types>
const std::string FormatString(const std::string& fmt_spec,
                               const Types&... args);
```

该函数实现如下：

```cpp
template <typename... Types>
const std::string FormatString(const std::string& fmt_spec,
                               const Types&... args) {
    std::stringstream builder;
    BuildFormatString(builder, fmt_spec, args...);
    return builder.str();
}
```

该函数声明一个用于构造字符串的 `stringstream`，然后通过调用辅助函数
`BuildFormatString` 完成最终的字符串格式化，之所以这样做的原因是可以把
`builder` 通过引用传递，从而不用在后续的递归中一次一次的声明临时
`stringstream`变量。`BuildFormatString` 使用递归的方式实现字符串的格式化，具体
实现如下。

```cpp
void BuildFormatString(std::stringstream& builder,
                       const std::string& fmt_spec) {
    builder << fmt_spec;
}

template <typename T, typename... Types>
void BuildFormatString(std::stringstream& builder,
                       const std::string& fmt_spec,
                       const T& first,
                       const Types&... args) {
    auto pos = fmt_spec.find_first_of("{}");
    if (pos > fmt_spec.length()) {
        builder << fmt_spec;
        return;
    }

    builder << fmt_spec.substr(0, pos);
    builder << first;
    BuildFormatString(builder, fmt_spec.substr(pos + 2), args...);
}
```

其中第一个普通函数（非模板函数）用来终止递归。这种实现方式只要参数支持
`stringstream` 的输出操作就可以正常的运行。这个版本并没有做过优化，比如它每
一次调用 `BuildFormatString` 都会产生一个 `std::string` 类型的临时变量用于存放
格式化说明符 `fmt_spec`。可以考虑通过增加一个 pos 参数来解决这个问题，或者是使
用 `const char*` 替代 `std::string` 作为格式说明符的类型。此外 `builder <<
fmt_spec.substr(0, pos)` 中的子串生成<del>可以通过循环取代</del>。**事实上直接
使用循环不会给性能带来任何的优化，因为字符的整块处理比循环单字符处理要快很多，
对于优化问题可以参考后面一小节**

# 测试结果

简单的测试代码如下：

```cpp
int main(int argc, char *argv[])
{
    cout << FormatString("", "hello", 3.14) << endl;
    cout << FormatString(" world!", "hello", 3.14) << endl;
    cout << FormatString("{} world!", "hello", 3.14) << endl;
    cout << FormatString("{} world! PI is {}", "hello", 3.14) << endl;
    cout << FormatString("{} world! PI is {}", "hello") << endl;
    return 0;
}
```

输出结果如下：

```cpp

world!
hello world!
hello world! PI is 3.14
hello world! PI is {}
```

如果 '{}' 占位符多于实际提供的参数，占位符会被保留，反之如果参数多于占位符，参
数会被忽略。

# 优化

**2016-03-24更新，之前这个版本没有优化过，后续出于性能调优的考虑，做了一点点小
的改动，记录在此。**

## 使用 `ostringstream` 替代 `stringstrem`

上一个版本个人疏忽，使用了 `stringstream` 作为字符串的`builder`，但是实际上我们
只是使用它作为输出缓冲区，所以可以直接使用 `ostringstream` 替代。

## `find_first_of` 返回值修正

`find_first_of` 在查找失败的疏忽返回的是 `string::npos` 通常这个值被定义为 `-1`
，但是因为它的类型 `string::size_type` 通常是一个无符号数，所以 `npos` 变成了能
表示的最大值，所以之前的 `pos > fmt_spec.length()` 判断才能正常工作，但是这并不
是可移植的方式，正确的写法应该是：

```cpp
if (pos == std::string::npos) {
}
```

## 使用 `ostringstream::write` 去掉前缀子串的创建

之前的版本中为了输入前缀子串使用了 `builder << fmt_spec.substr(0, pos);` 这样的
语句，它使得每一次前缀输入都需要生成一个前缀子串。我们可以使用 `write` 成员函数
去掉这个子串的创建，直接写入 `string` 中的一块内容。

```cpp
builder.write(fmt_spec.data(), pos);
```

## 使用位置下标去掉后缀子串的创建

在之前处理完一个参数之后，通过递归的方式处理下一个参数：

```cpp
BuildFormatString(builder, fmt_spec.substr(pos + 2), args...);
```

这个递归调用的第二个参数会产生一个临时变量，这个变量其实可以通过位置下标去掉，
做法是更改函数的签名，让他接受一个额外的参数如下：

```cpp
template <typename T, typename... Types>
void BuildFormatString(std::ostringstream& builder, const std::string& fmt_spec,
                       std::string::size_type idx, const T& first,
                       const Types&... args);
```

查找的占位符的时候使用带位置信息的版本

```cpp
auto pos = fmt_spec.find_first_of("{}", idx);
```

当然写入前缀的后缀处理的时候也要相应的更改

```cpp
builder.write(fmt_spec.data() + idx, pos - idx);
BuildFormatString(builder, fmt_spec, pos + 2, args...);
```

通过上面的这些更改，性能得到了一定的提升，对于下面的测试：

```cpp
const int kOutLoopCount = 1000;
const int kInnerLoopCount = 1000000;

int main(int argc, char *argv[])
{
    cout << FormatString("") << endl;
    cout << FormatString("world") << endl;
    cout << FormatString("{} world!", "hello", 3.14) << endl;
    cout << FormatString("{} world! PI is {}", "hello") << endl;
    cout << FormatString("{} world! PI is {}", "hello", 3.14) << endl;

    long sum = 0;
    for (int i = 0; i < kOutLoopCount; ++i) {
        auto beg = high_resolution_clock::now();
        for (int i = 0; i < kOutLoopCount; ++i) {
            FormatString("{} world! PI is {}", "hello", 3.14);
        }
        auto end = high_resolution_clock::now();
        sum += (end-beg).count();
    }
    cout << "average: " << sum / kOutLoopCount << endl;

    return 0;
}
```

没有优化过的版本，平均实践大概是 `1294218` 优化过的版本的平均时间大概是
`1068972`，效果还是不错的。

[^1]: `std::string` 在实现的时候没有考虑过多态的使用，比如它没有声明虚拟析构函
数，所以不建议继承自 `std::string` 扩展自己的字符串，你可以使用组合替代继承来
扩展标准库中的字符串，但是使用全局函数的方式更简单方便，所以这里采用这种方式。
