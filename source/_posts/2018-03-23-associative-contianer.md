---
title: C++轮子——STL关联容器
date: 2018-03-23 21:05:36
tags:
 - C/CPP
 - C++轮子
---

上一篇文章中我们简单的介绍了一下STL中的序列容器和容器适配器，这篇文章中我们将重点介绍STL中的关联容器（最后两个在概念上应该不是关联容器，但是因为和前面的容器联系太紧密，统一放在这里讲解），主要内容包括：

- std::set
- std::map
- std::multi_map
- std::multi_set
- std::unordered_map
- std::unordered_set

<!--more-->

# std::set

通常提到管理容器，大家可能首先会想到的是`std::map`，这里先讲解`std::set`是避免出现`关联数组`和`关联容器`概念上的混淆。前者是数据结构中的概念，而后则是C++中特有的概念。

## 什么关联容器

关联容器这个概念，在C++里面是一个concept（关于concept的讨论我们在上一篇文章中有讲解，这里不再赘述），它的基本定义如下：

> An AssociativeContainer is an ordered Container that provides fast lookup of objects based on keys.

翻译成中文，大概是说管理容器是提供了基于key的快速访问的有序容器。它实际上是 Container 这个 concept 的 refinement。

## Refinement

Refinement 在泛型编程中是一个非常重要的概念，它和 concept 的关系类似于OO世界中父子集成关系。我们知道 Concept 表示的不是一个类型而是一个类型集合，所以 Refinement 表示的其实也是一个类型集合，这些类型比 Concept 中的定义的类型要有更多的约束和特性（约束和特性其实是一个问题的两个方面，你约束的越是严格，你能够使用的特性越多，但是你能使用的场景越少，比如 Java 中 Object 类哪儿都能用，又哪儿都用不了）。

## 有序

在数学概念上，集合表示具有某种特定性质的事物的总体，它有三个特性：无序性、互异性、确定性。

所以个人认为，C++世界里面的`std::set`其实并不是传统意义上的集合，因为它不符合无序的特性（当然你也可以理解成无序表示任何顺序都可以，有序也属于无序的一种状态）。

需要特别提醒的是，因为关联容器使用key值排序，你不能修改key值，否则你会破坏关联容器的内部结构（虽然有些情况下更改key值在逻辑上是合理并且合法的【1】）。在C++11之后，关联容器返回的迭代器，key 都默认是 const。如果你需要更改 key 的值，最好的方式是先删除，再插入（有 move 的存在，这一步的开销相对小一些）。

## 互异

互异是指集合内部的任意两个元素都不相同，这一点在`std::set`中满足，但是在`std::multi_set`中并不满足。

`std::set`元素的互异在编程中有一个非常大的作用是去重，当然你也可以使用sort+unique的方式来处理去重，但是`std::set`的方式在逻辑上更具有表达力。

## 成员方法

`std::set`中的很多成员和`std::map`类似，只不过它们的`value_type`不一样，`std::set<std::string>`的`value_type`是`std::string`，而`std::map<int, std::string>`的`value_type`是`std::pair<int, std::string>`。因此这一小节不介绍相关的成员函数，你可以查看`std::map`的相关成员函数的介绍，然后对照`std::set`的[文档](http://devdocs.io/cpp/container/set)来理解这一部分内容。

# std::map

`std::map`可能是C++世界使用得最广泛的关联容器，它以键值对的形式存储数据的一种关联容器。

## 关联数组

在数据结构的范畴里面，map又被称为关联数组。一个普通的数组，可以使用int（严格来说应该是size_t）做为下标来访问数组内部的元素，关联数组则可以使用任意类型做为下标来存取数组内部的数组。当然这里说的数组是一个逻辑上的概念，关联数组的实现物理上的实现可能是树或者哈希表等等。

需要注意的是关联数组和关联容器是完全不同的两个概念，关联容器强调的是提供快速访问的有序容器，这个概念只存在于C++中；而关联数组强调的是键值对的形式存储数据，并通过key来快速访问数据，这个概念是传统数据结构中的概念，在不同的语言对应不同的类型，比如Java的map，Python的dict。关联容器不一定是关联数组，比如set；关联数组也不一定是关联容器，比如unordered_map。

## 红黑树

map和set的实现通常是一个颗红黑树，红黑树是一种平衡二叉排序树。它的节点要么是黑要么是红（我也很好奇为什么不是要么黑要么白），但是如果节点是红，它的两个子节点一定是黑，而且任意节点到叶子节点的任意路径中的黑色节点一样多。这个特性可以保证树的两个分支深度最多只会出现2:1的关系（全黑和红黑相间），所以它基本平衡，它不完美，没有AVL好看，但是它的实现比较简单，出现不平衡的时候，调整比较简单，所以在实际运用中使用的频率很高（比如Linux内核的进程调度用的就是红黑树）。

# 数据的添加

往map里面添加数据有三种方式：operator[]，insert，emplace。效率上来说，通常后面的效率会高于前面的效率。emplace 和 insert 的区别我们在[C++轮子 —— STL 序列容器](/2018/02/25/stl-part-two-vector/#push-back-vs-emplace-back)一文中有提到过，这里不再重复解释。

理论上来说，operator[]实际上是数据访问的方式，但是因为它返回的是引用，所以可以直接用于改变元素的值。如果 key 不存在，operator[]会先调用`mapped_type`的默认构造创建一个空的对象（如果你确保数据存在，可以考虑使用std::map::at，如果key不存在它会抛异常）。比如一个统计单词出现次数的程序可以这样写：

```
std::map<std::string, std::size_t> words;

std::istringstream iss(str);
std::string word;
while (iss >> word) {
    words[word]++;
}
```

上面这段代码之所以能够成功运行，是因为如果单词第一次出现，words[word]会使用std::size_t的默认构造（std::size_t()）创建一个新的元素并返回，这样一来新的单词出现次数可以做正确的累加。

同样因为这个特性你不应该使用operator[]来判断数据是否存在，比如你不能像下面这样去判断一个元素是否存在。

```c++
if (words["word_not_exist"]) {
    //
}
```

因为如果元素不存在，它会创建一个新的对象，如果你只是想判断元素是否存在，这种行为通常都不是你想要的。

你实际上需要的是功能是`Contains`，正确的写法是使用`std::map::find`函数：

```
auto iter = words.find("word_not_exist");
if (iter != std::end(words)) {
    //
}
```

operator[]在mapped_type是一个比较重的类型的时候，也可能造成不必要的开销。因为它会县构建一个空的对象，随后又被特换掉，比如你有一个map用于映射人名和职位名称。

```
std::map<std::string, std::string> employee;
employee["bob"] = "software enginer";
```

这个地方先用"bob"构造了std::string，然后用这个string作为key创建了一个空字符串作为value，然后返回这个vlaue的引用，最后调用这个value的operator=完成赋值。实际上中间的空的字符串完全可以不需要构造，我们可以直接通过调用`insert`方法来插入数据。

```
employee.insert(std::make_pair("bob", "software enginer"));
```

需要注意的是map的value_type是std::pair<std::string, std::string>而不是std::string. 如果你不想构建这个pair，你可以使用`emplace`。

```c++
employee.emplace("bob", "software enginer");
```

## find

前面在谈到`Contains`的时候我们有提到，std::map::find，之所以单独提到它是因为序列容器的查找通常使用的都是算法：std::find，而关联容器通常都提供`find`成员，你当然也可以用std::find，但是它的复杂度是O(N)，而`find`成员的复杂度是O(logn)。通常如果一个容器提供了某个算法的同名成员函数，你都应该应该优先使用成员函数，因为它们的效率更高，获取无法使用算法实现（比如 std::map::count，std::list::erase等）。当然换句话说，如果你找不到某个你想要的成员函数的时候，你可以去算法库里面找找。

## lower_bound，upper_bound，equal_range

关联容器的核心特点是元素有序，所以可以快速的查找数据。它不仅可以查找数据是否存在，它还可以用于查找给定元素的上下界区间。你可以使用lower_bound函数查找到给定元素的下界，也就是容器中第一个不小于指定元素的位置；同样你可以使用upper_bound来找出到给定元素的上界，也就是容器中第一个大于给定元素的位置；而equan_range可以用于返回前两者构成的区间。

定义上可能比较绕，我们举一个具体的例子：

```
const std::map<uint32_t, uint32_t> kBitratesOfResolution = {
        {192 * 144, 400},       // 144p
        {320 * 240, 500},       // 240p
        {480 * 360, 600},       // 360p
        {640 * 480, 900},       // 480p
        {1280 * 720, 1800},     // 720p
        {1920 * 1080, 30000},   // 1080p
};

kBitratesOfResolution.lower_bound(1280*720);    // 720p 指定的那个位置
kBitratesOfResolution.upper_bound(1280*720);    // 1080p 所指的那个位置
kBitratesOfResolution.equal_range(1280*720);    // [720p, 1080p]
```

如果你在你的程序中需要查找最吻合的某个值（比如上面例子中的找到给定分辨率最合适的比特率），你可以使用上面提到的这几个函数，它们可以在logn之内找到给定的位置。如果你使用的是std::vector，你可以考虑使用 sort + binary_search 来实现这个功能。

# std::multi_set 和 std::multi_map

multi_set 和 multi_map 在接口和实现结构上和 set 和 map 是一样的，只不过在
multi_xxx 版本支持 key 值重复。key 值重复对于类似于字典这种功能比较有用（一个字可以有多个解释）。

# std::unordered_map、std::unordered_set、std::unordered_multimap、std::unordered_multiset

这四个容器在功能上和map、set类似，但是通常都使用哈希表来实现的，这四个容器是在C++11中才加入到标准库中的，但是实际上在早加入标准之前在很多STL的实现中就默认带了类似功能的容器，通常的命名是 hash_map 和 hash_set。为了避免和之前的内容产生冲突，这四个个容器在加入标准库的过程中被命名为以`unordered_`开头。

这几个容器在接口和内部结构上基本上是一致的，因此下面的讲解以unordered_map为例，其他几个大家可以对照文档理解。

## 无序关联容器

这几个容器正如名字上显示的那样是无序的，所以他们不是关联容器，他们在容器分类上被称为无序关联容器。下面是 UnorderedAssosiativeContainer 的简明定义。

> Unordered associative containers are Containers that provide fast lookup of
> objects based on keys. Worst case complexity is linear but on average much
> faster for most of the operations.

## 哈希表

《程序设计实践》一书中提到四个最常用的数据结构：数组、链表、树、哈希表。在STL中分别对应vector，list，map，unordered_map。标准虽然不规定unordered_map的实现方式，但是通常的实现都是哈希表。

哈希表的典型结构如下图：

```
                                         table
                                         +------+
                                         |      |
                                         |      |
                                         |      |
                                  +----> |      |
                                  |      +------+
                                  |      |      |
                                  |      |      |
                                  +----> |      |
                                  |      +------+
                                  |      |      |     +-----+-----+-----+
              +---------------+   |      |bucket+---> |     |     |     |
              |               |   +----> |      |     +-----+-----+-----+
E  +--------> | hash function +---+      +------+
              |               |   |      |      |
              +---------------+   +----> |      |
                                  |      |      |
                                  |      +------+
                                  |      |      |
                                  +----> |      |
                                  |      |      |
                                  |      +------+
                                  |      |      |
                                  +----> |      |
                                  |      |      |
                                  |      +------+
                                  |      |      |
                                  |      |      |
                                  +----> |      |
                                         |      |
                                         +------+

```

一个元素，通过一个 hash 函数，映射到一个哈希表的一个 bucket（也有人称之为slot） 中。理想情况下，对于哈希表的基本操作都可以在常量时间内完成。所以在查找元素方面，hash 表的效率是最高的。

## 哈希冲突

和红黑树不同的是，哈希表的时间复杂度是一种概率论上的值（计算方法可以参考《算法导论》一书），实际的效果到底怎么样，还要看元素在哈希表中是分布情况。

首先需要注意的是，元素和bucket之间，不会是一一对应的关系，如果有这种关系，我们直接使用数组就可以了，不用使用哈希表。我们给定一个哈希函数，随着元素的增多一定会产生哈希冲突，解决哈希冲突的方式比较多，STL中通常使用外链的方式，把所有哈希值相同的元素通过一个单链表串起来。哈希冲突越严重，哈希表的效率就越低。假如我们给出下面这样一个哈希函数：

```
std::size_t hash_fun(int a) {
    return 0;
}
```

那么所有的元素都会被放到第一个bucket的外链中，这样一来哈希表也就退化成了一个单链表链表，它的操作是非常低效的。所以哈希表的效率关键在于选择一个好的哈希函数（如何选择好的哈希函数是一个非常复杂的问题，有大量的算法，有兴趣的同学可以去看看《算法导论》）。

## 哈希函数

unordered_map的原型如下：

```
template<
    class Key,
    class T,
    class Hash = std::hash<Key>,
    class KeyEqual = std::equal_to<Key>,
    class Allocator = std::allocator< std::pair<const Key, T> >
> class unordered_map;
```

可以看出，默认情况下它使用 std::hash 计算哈希值。但是 std::hash 实际上是一个函数，而是仿函数（重载了函数调用操作符的类），它的原型如下：

```
template< class Key >
struct hash;
```

注意这个模板本身没有定义，它只有声明，因为没有任何一个默认实现是适合所有类型的。标准库为内置类型定义了这个模板的特化：

```
template<> struct hash<bool>;
template<> struct hash<char>;
template<> struct hash<signed char>;
template<> struct hash<unsigned char>;
template<> struct hash<char16_t>;
template<> struct hash<char32_t>;
template<> struct hash<wchar_t>;
template<> struct hash<short>;
template<> struct hash<unsigned short>;
template<> struct hash<int>;
template<> struct hash<unsigned int>;
template<> struct hash<long>;
template<> struct hash<long long>;
template<> struct hash<unsigned long>;
template<> struct hash<unsigned long long>;
template<> struct hash<float>;
template<> struct hash<double>;
template<> struct hash<long double>;
template< class T > struct hash<T*>;
```

也就是说，使用上面这些类型作为unordered_map的key的时候可以直接使用，但是其他类型会导致编译错误，因为找不到std::hash的实现。假如我们想要使用我们自己定义的类型作为key，那么我们要实现自己的std::hash的特化。


```c++
struct S {
    std::string first_name;
    std::string last_name;
};

bool operator==(const S& lhs, const S& rhs) {
    return lhs.first_name == rhs.first_name && lhs.last_name == rhs.last_name;
}

namespace std
{
    template<> struct hash<S>
    {
        typedef S argument_type;
        typedef std::size_t result_type;
        result_type operator()(argument_type const& s) const
        {
            result_type const h1 ( std::hash<std::string>{}(s.first_name) );
            result_type const h2 ( std::hash<std::string>{}(s.last_name) );
            return h1 ^ (h2 << 1);
        }
    };
}

std::unordered_map<S> names;
```

特化在泛型中的地位，类似于虚函数在OO中的地位，一个对应静多态，一个对应动多态。你也可以直接实现另一个仿函数，作为unordered_map的参数传入进去。

```
// custom hash can be a standalone function object:
struct MyHash
{
    std::size_t operator()(S const& s) const
    {
        std::size_t h1 = std::hash<std::string>{}(s.first_name);
        std::size_t h2 = std::hash<std::string>{}(s.last_name);
        return h1 ^ (h2 << 1);
    }
};

std::unordered_map<S, MyHash> names;
```

哈希函数的实现有下面两个细节需要注意：

1. 实现 operator==，哈希表要求两个相等的元素，哈希值必须一致，通常实现了函数函数的同时也会实现判等操作符。

2. 哈希函数的实现可以借助 boost::hash_combine 来完成，具体参考 [boost::hash](http://www.boost.org/doc/libs/1_66_0/doc/html/hash/reference.html#boost.hash_combine) 库。

## 装载因子

哈希表是一种典型的用空间换时间的数据结构，我们说哈希表的效率关键在于哈希函数取得的散列效果，哈希值越是分散，哈希表的效率越高。哈希函数的散列效果则和装载因子相关。

所谓装载因子是指：size()/bucket_size()，可以看出装载因子越高（一般来说这个值是 0.75，虽然标准并不规定这个值），哈希冲突的概率越大，装载因子越低，哈希冲突的概率越小，但是浪费的空间也越多。

你可以通过 std::unordered_map::load_factor 获取目前的装载因子，通过
std::unordered_map::max_load_factor 来获取或者设置装载因子。随着元素的增加，load_factor 会不断接近 max_load_factor()，当它超过这个后者的时候，unordered_map会自动调整内存 bucket，并重新做散列，以降低 load_factor，保证效率。如果你的空间足够的多可以考虑增大max_load_factor来提高效率。

当然这个自动重散列的过程和 std::vector 的自动扩充空间一样，会导致数据的拷贝，效率上比较低。对于 std::vector 如果你知道你的数据有多少，你可以通过 reserve 来预留空间，对于 unordered_map 来说，你可以使用 `reserve` 和 `rehash` 两个函数来完成这件事情。这两个函数的关系在于前者接收的是元素个数，后者接收的bucket的个数。假如你预计你有 100 个元素，你可以通过下面两种方式预留空间。

```
reserve(100);
reserve(100 / max_load_factor());
```

## bucket

std::unordered_map 中其实还存在一系列用于处理 bucket 的接口，我平时没有用过，并不太清楚他们的使用场景，但是他们对于打印哈希表的内部结构比较有用

---

【1】 《Effctive STL》中有提到，如果key是对象，你只是使用这个对象中的某个成员变量当成实际的key，更改其他成员是合法的。但是个人还是不建议这么使用。
