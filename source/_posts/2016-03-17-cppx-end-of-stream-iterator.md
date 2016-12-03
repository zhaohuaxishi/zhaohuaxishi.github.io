title: C++11 中使用 {} 作为串流迭代器 EOF
date: 2016-03-17 09:53:36
tags:
 - C/CPP
---

`stream iterator` 是一种神奇的迭代器适配器，它让我们可以使用 `STL` 算法操纵输
入输出流，比如把 `vector` 中的内容全部输出，可以简单的写成这样。

```cpp
vector<int> coll = {1, 2, 3, 4, 5, 6};
copy(coll.cbegin(), coll.cend(), ostream_iterator<int>(cout, " "));
```

当然你也可以使用这种方式处理输入，比如从标准输入中输入元素到 `vector` 中去。

```cpp
vector<int> coll;
copy(istream_iterator<int>(cin),    // start of source
     istream_iterator<int>(),       // end of source
     back_inserter(coll));          // destination
```

`istream_iterator` 要比 `ostream_iterator` 难用一些，因为你必须要使用一个
`EOF` 表示输入的结束，这个概念可以用 `istream_iterator` 的默认构造函数产生的对
象表示。也就是上面这个 `istream_iterator<int>()`。

这种语法看起来比较诡异，我在看 《C++ 标准库》[^1]这本书的勘误的时候发现了一种新
的语法，直接使用`{}`。

```cpp
copy (istream_iterator<string>(cin), // start of source
      {},                            // end of source
                                     //(default constructed istream iterator)
      back_inserter(coll));          // destination
```

之所以可以这样做主要归功于 `C++11` 统一了初始化语法，可以在任何地方使用 `{}`
来初始化，在 `copy` 中，因为我们可以根据第一个参数推导出第二个参数的类型，所以
可以直接通过 `{}` 来调用默认的构造函数创建代表 `EOF` 的迭代器。

[勘误][errata]的原话如下：

> Note that since C++11, you can pass empty curly braces instead of a default
> constructed stream iterator as the end of the range. This works because the
> type of the argument that defines the end of the range is deduced from the
> previous argument that defines the begin of the range:

另外需要提醒的是，《C++ 标准库》的中文译本应该是基于第一次印刷版本翻译的，原书
有一些错误，在英文版的[勘误][errata]中有修正，大家如果读这本书应该关注一些它的
勘误，以免被一些错误所误导。

[errata]: http://www.cppstdlib.com/errata.html

[^1]: 我读的是第二版，包含最新的`C++11`标准库内容
