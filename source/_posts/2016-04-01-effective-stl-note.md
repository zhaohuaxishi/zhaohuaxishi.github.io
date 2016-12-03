title: 读书札记之 —— 《Effective STL》
date: 2016-04-01 19:28:21
tags:
 - C/CPP
 - 读书笔记
---

`Scott Meyers` 的书，似乎每一本都值得一读再读，这本书我是第二次读。上次读这本书
基本上是在车上读完的，那时候在公司实习，上下班的路上读完这本书，不厚，但是很值
得人回味。

那时候对于`STL`并不理解，至少比现在理解的要少一些，所以很快就忘了书里说了些什么
。以至于后来看到《深入理解C++11》这本书引用《Effective STL》一书的内容的时候没
有半点印象，这也是我打算重读这本书的原因。

看这本书之前，我看了《C++ 标准库》这本砖头书，对于标准库中的各个组件有了更加深
刻的理解，所以再次看这本书的时候，感悟和第一次读的时候有很大的区别。

<!--more-->

# 书中有一些信息已经过时了

虽然我是作者的脑残粉，不过我还是得承认，这本书中有些信息真的已经过时了。我简单
的罗列了一下：

- 容器中的数据必须能够正常拷贝，C++11 中 `move` 语义的出现，去掉了这一限制。（
  item 3）

- `string` 的引用计数。C++11 中不再允许使用引用计数的方式实现 `string`。（item
  15）

- 不要修改 `set` 或 `multiset` 中的键。C++11 中不允许我们直接修改 set 的键，书
  中提到是做法现在来说不合法。（item 22）

- 优先使用 `iterator`。在 C++11 中由于 cbegin，cend 的出现，你应该优先使用
  `const_iterator`，这一点在作者的新书 《Effective Modern C++》中有提到。（item
  26）

- ptr_fun, mem_fun, mem_fun_ref, functor 适配等。在新的标准中，这些东西已经被废
  弃了。我们应该优先考虑使用 `binder`，`function`，它解决了上面这些组件原本解决
  的问题，并且是以一种更加优雅的方法。（item 41）

还有一些信息，没有过时，不过有更优雅的方式完成。

- 使用括号来修正下面代码被解析成函数声明（item 6）

    ```
    list<int> data(istream_iterator<int>(dataFile),
   	           istream_iterator<int>())

    ```

  在C++11中可以直接使用`{}`作为第二个参数，从而避免这个问题。

- 使用 swap 技术去掉容器中多余的元素。C++11中，`vector`，`string`，`deque`都提
  供了`shrink_to_fit`函数完成这一功能。（item 17）

# 很有深度的一本书

不，虽然我说这本书有些信息过时了，但我没有说这本书不值得一读。这仍然是我见过最
有深度的STL的书（当然，你可以鄙视我，因为我一共也没有看过几本），非常值得每一个
C++程序员去读一读。

# STL 的坑

这本书提到了很多STL的坑，下面是你需要重点注意的一些大坑。

- 确保目标区间足够大（item 30）

    ```
    transform(v.begin(), v.end(),
    	      result.end(),
	      transmogrify)
    ```

  这段代码无法工作，因为 result.end() 不是一个合法的插入地址，你应该考虑使用
  back_inserter

- remove 不会删除元素（item 33）

    ```
    remove(v.begin(), v.end(), 99);
    ```

  这段代码不会从容器中伤处任何的东西。你需要对返回值调用`erase`。

- 不要在判别式（有是书中翻译成谓词）改变状态（item 39）

    ```
    class BadPredicate {
    public:
        BadPredicate() : timeCalled(0) {}
        bool operator(const Widget&) {
            return ++timeCalled == 3;
        }

    private:
        int timeCalled;
    }
    ```

  上面这段代码是一段很拙劣的代码。
