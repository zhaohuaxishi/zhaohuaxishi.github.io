---
title: C++ 常用库 —— JSON for Modern C++
date: 2017-05-14 19:48:29
tags:
 - C/CPP
---

`JSON`因该算是目前网路数据交换格式的事实标准，似乎没有那一种语言不存在支持这种数
据格式的库，C++也不例外。不过`JSON`虽然和语言无关，但是它毕竟源于动态语言，所以
在C++中，很多`JSON`库接口都不太自然。`C++11`引入的`universal initialization`和`
initializer list`让一个拥有自然的接口的`JSON`库成为可能。这篇文章介绍的就是这样
一个库——[JSON for Modern C++][json]。我第一次见到这个库的时候有了使用动态语言的
感觉。

 [json]: https://github.com/nlohmann/json

```c++
json array = {"hello", 1, 2.5, false, true, {1, 2}};
```

你没有看错，上面写的真的是`C++`的代码不是`Python`。这篇文章会介绍关于这个神奇的
`JSON`库的一些使用和实现上的细节，一起看看吧。

<!--more-->

# JSON 的基础知识

下面这段话来自`JSON`的[官方文档][jsonorg]：

 [jsonorg]: http://www.json.org/json-zh.html

> JSON建构于两种结构：
>
> - “名称/值”对的集合（A collection of name/value pairs）。不同的语言中，它被理
>   解为对象（object），纪录（record），结构（struct），字典（dictionary），哈希
>   表（hash table），有键列表（keyed list），或者关联数组 （associative array）
>   。
> - 值的有序列表（An ordered list of values）。在大部分语言中，它被理解为数组（
>   array）。

另外一个比较重要的点是，`JSON`中的值可以是下面几种：

```
string, number, true, false, object, array, null
```

结合上面的定义你会发现，这是一种递归的定义，比如数组是值的有序列表，而值又可以是
数组，所以这是一种无穷的结构。

# JSON 值在静态语言中的表示 —— nlohmann::json

前面提到一个`JSON`值可以是`string, number, true, false, object, array, null`中的
任意一种，但是C++是一种静态类型语言，任何值都有它固定的类型，一个值不能既是int又
是double。

解决这个问题的关键点在于抽象，也就是《C++沉思录》中反复提到的一个概念——用类抽象
一个概念。`nlohmann::json`这个类抽象的概念就是`JSON`的值。也就是说从静态语言的角
度考虑问题，所有这些值都有同一个类型：`nlohmann::json`：

```c++
json string_value = "The quick brown fox jumps over the lazy dog.";
json number_value = 1;
json boolean_value = true;
json array_value = {1, 2, 3, 4};
json object_value = {{"age": 21}};
```

当然因为`json`本身也是一个类型，所以我们可以声明一个存放`json`的array和object。

```c++
json json_array = {
    json(1),
    json("The quick brown fox jumps over the lazy dog."),
    json(true),
    json({1, 2, 3, 4})
};

json json_object = {
    {"object": {
            {"subobject": 1}
        }
    }
};
```

这样一来就变成了值的递归了，可以扩展成为一个无穷的结构。

*需要注意的是，上面例子中显式的构造 json 对象是没有必要的，因为这些构造函数都不
是 explicit 构造函数，支持隐式转化，上面这样写只是为了方便问题的说明。*

## JSON 值的实际类型

当然无论我们如何抽象，它终究无法逃离C++作为一种静态类型语言的核心，我们可以让
`json` 类即表示`number`又表示`array`但是在底层的实现上我们终究无法让一个变量的类
型动态的改变，即便现在 C++11 支持 `auto` 也是如此。

```c++
auto json_value = 0;
json_value = "string";
```

上面这种语法在目前的 C++ 中是不合法的。在[JSON for Modern C++][json]的底层实现上
，它为`JSON`中的每一种`value`类型都设定了对应的C++类型，其中默认值如下：

```
- object:       std::map
- array:        std::vector
- string:       std::string
- number:       std::uint64_t 和 std::int64_t 和 double
- true, false:  bool
- null:         nullptr_t
```

注意上面的 `number` 使用了三个不同的值是因为在[JSON for Modern C++][json]内部
`number`其实被细分成了`number_integer`、`number_unsigned`、`number_float`三种。

我们看到的`nlohmann::json`这个类型实际上是一个别名：

```
using json = basic_json<>;
```

也就是说它实际上是模板类`basic_json<>`使用默认的模板参数时实例化出来的类型。而
`basic_json`这个模板类的声明如下：

```c++
template <
    template<typename U, typename V, typename... Args> class ObjectType = std::map,
    template<typename U, typename... Args> class ArrayType = std::vector,
    class StringType = std::string,
    class BooleanType = bool,
    class NumberIntegerType = std::int64_t,
    class NumberUnsignedType = std::uint64_t,
    class NumberFloatType = double,
    template<typename U> class AllocatorType = std::allocator,
    template<typename T, typename SFINAE = void> class JSONSerializer = adl_serializer
    >
class basic_json;
```

这个模板类有9个模板参数，其中前面7个就是用于表示`JSON value`的实际类型。
`ObjectType`和`ArrayType`这两个模板参数的语法叫做`模板的模板参数`也就是说，
`ObjectType`这个模板参数匹配的实际是另一个模板，这个模板至少有`U, V`两个模板参数
和`Args`表示的其他可选模板参数，需要注意的是，`U, V, Args`对于 `basic_json` 来说
是没有意义的，因为它 `ObjectType` 的模板参数，不是 `basic_json` 的模板参数，语法
上可以省略它们（这和函数的声明可以省略名字相通）。

实际上如果你愿意，你可以替换掉其中的一些类型，比如你想要用`std::deque`表示
`array`那么你可以定义你自己的类型`JSON`类型如下：

```
using MyJson = nlohmann::basic_json<std::map, std::deque>;
```

不过目前来看，我们似乎没有办法使用`std::unordered_map`作为`ObjectType`，这好像是
一个库本身的问题[std::unorderd_map cannot be used as ObjectType #164][issue]，

 [issue]: https://github.com/nlohmann/json/issues/164

在 `basic_json` 中值的实际类型根据模板参数被定义成了下面这几种：


```c++
using object_t = ObjectType<StringType,
      basic_json,
      std::less<StringType>,
      AllocatorType<std::pair<const StringType,
      basic_json>>>;
using array_t = ArrayType<basic_json, AllocatorType<basic_json>>;
using string_t = StringType;
using boolean_t = BooleanType;
using number_integer_t = NumberIntegerType;
using number_unsigned_t = NumberUnsignedType;
using number_float_t = NumberFloatType;
```

## 类型判断

`nlohmann::json`提供了下列这些借口来判断内部实际的值是什么类型。

```
is_primitive , is_structured , is_null , is_boolean , is_number ,
is_number_integer , is_number_unsigned , is_number_float , is_object , is_array
, is_string
```

其中 `is_structured` 是指 `is_object() or is_array()`，`is_primitive` 则是指
`is_null() or is_string() or is_boolean() or is_number();
`

在 `basic_json` 内部有一个成员变量：`m_type`，它的类型是一个枚举类：

```c++
enum class value_t : uint8_t
{
    null,            ///< null value
    object,          ///< object (unordered set of name/value pairs)
    array,           ///< array (ordered collection of values)
    string,          ///< string value
    boolean,         ///< boolean value
    number_integer,  ///< number value (signed integer)
    number_unsigned, ///< number value (unsigned integer)
    number_float,    ///< number value (floating-point)
    discarded        ///< discarded by the the parser callback function
};
```

其中 `discarded` 这种类型只存在与解析的过程中，实际的 `json` 值不会是这种类型。

## 值的存储

在 `basic_json` 定义了一个成员变量`m_value`，用于存储实际的值，为了能够让
`m_value`可以表示多种类型的数据，它的类型被定义成了一个`union`。

```c++
union json_value
{
    object_t* object;
    array_t* array;
    string_t* string;
    boolean_t boolean;
    number_integer_t number_integer;
    number_unsigned_t number_unsigned;
    number_float_t number_float;

    // 其他一些构造函数
};
```

我们都知道 `union` 的`size`是由`size`最大的那个成员变量决定的，为了能够节省空间
，`object`, `array`, `string` 这三种类型的之实际上是使用指针来存储的。

# 类型的自动转换

如果单说`JSON`值的抽象，一个中等水平的程序员都可以做到，[JSON for Modern
C++][json]真正厉害的地方在于它的自动转换，正是这些自动转换让我们觉得它的API自然
而顺手。

## C++ 中的自动类型转换

先说一下自动类型转换，在C++中，类型之间的自动转换分为两种：

1. 非 `explicit` 的单参数构造函数（逻辑上单参即可，多参数但是后面都有默认参数也
   可以），它可以用于把其他类型自动转换成本类型，比如我们最常见的 `const char*`
   到 `std::string` 转换。

   ```
   std::string hello = "hello";
   ```

2. `operator Type()`操作符，这种函数可以用于把本类型自动转换成`Type`类型。比如下
   面这个例子：

   ```
   class Widget {
   public:
       Widget(const std::string& name) : name_(name) {}

       operator std::string() {
           return name_;
       }

   private:
       std::string name_;
   };

   Widget widget = "btn";
   std::string btn = widget;
   ```

## 数据自动转换为 `nlohmann::json`

如前所述，这是通过非 `explicit` 单参构造函数实现的，`basic_json` 定义了很多这一
类的构造函数，其中最强大的一个莫过于下面这个：

```c++
template<typename CompatibleType, typename U = detail::uncvref_t<CompatibleType>,
         detail::enable_if_t<not std::is_base_of<std::istream, U>::value and
                             not std::is_same<U, basic_json_t>::value and
                             not detail::is_basic_json_nested_type<
                                 basic_json_t, U>::value and
                             detail::has_to_json<basic_json, U>::value,
                             int> = 0>
basic_json(CompatibleType && val) noexcept(noexcept(JSONSerializer<U>::to_json(
            std::declval<basic_json_t&>(), std::forward<CompatibleType>(val))))
{
    JSONSerializer<U>::to_json(*this, std::forward<CompatibleType>(val));
    assert_invariant();
}
```

这个构造函数体现了作者极强的模板功底，让人叹服，如果去掉各种细节实际上这个函数调
用了`JSONSerializer<U>::to_json`来完成构造。`JSONSerializer`是`basic_json`的一个
模板参数，默认情况下是`adl_serializer`，`ADL`是一个非常重要的C++术语，指的是通过
参数的命名空间查看函数的。

对于 `nlohmann::json` 指定的值类型（或者可以自动转换为指定的类型），都会有一个
`to_json` 函数的重载，比如对于`bool`有如下定义：

```c++
template<typename BasicJsonType, typename T, enable_if_t<
             std::is_same<T, typename BasicJsonType::boolean_t>::value, int> = 0>
void to_json(BasicJsonType& j, T b) noexcept
{
    external_constructor<value_t::boolean>::construct(j, b);
}
```

更厉害的是，对于那些不是指定的类型(std::map, std::vector, std::string 等)，它内
部定义了另一个匿名名字空间中的全局变量`to_json`

```c++
constexpr const auto& to_json = detail::static_const<detail::to_json_fn>::value;
```

去掉各种细节，最终他实际上是`to_json_fn`这个类的一个变量，这个类重载了函数调用操
作符。

```c++
struct to_json_fn
{
  private:
    template<typename BasicJsonType, typename T>
    auto call(BasicJsonType& j, T&& val, priority_tag<1>) const noexcept(noexcept(to_json(j, std::forward<T>(val))))
    -> decltype(to_json(j, std::forward<T>(val)), void())
    {
        return to_json(j, std::forward<T>(val));
    }

    template<typename BasicJsonType, typename T>
    void call(BasicJsonType&, T&&, priority_tag<0>) const noexcept
    {
        static_assert(sizeof(BasicJsonType) == 0,
                      "could not find to_json() method in T's namespace");
    }

  public:
    template<typename BasicJsonType, typename T>
    void operator()(BasicJsonType& j, T&& val) const
    noexcept(noexcept(std::declval<to_json_fn>().call(j, std::forward<T>(val), priority_tag<1> {})))
    {
        return call(j, std::forward<T>(val), priority_tag<1> {});
    }
};
```

这个类的设计也是令人拍案叫絕的，首先`call`这个函数的优先级分派上`priority_tag`的
设计非常精妙：

```c++
template<unsigned N> struct priority_tag : priority_tag < N - 1 > {};
template<> struct priority_tag<0> {};
```

因为子类可以自动转换为父类，所以匹配上`priority_tag<1>`优先级高，但是如果不成功
会有自动调用`priority_tag<0>`，这里再通过 static_assert 在编译期间给出可读的错误
信息，实在让人叹为观止。

当然这些都不是重点，第一个 `call` 函数会通过 `ADL` 查找到位于 T 类型同一
`namespace`下面的 `to_json` 函数。也就是说用户定义的任何类型，只要在同一
`namespace`下面实现`to_json`就可以自动转化为 `basic_json`。每次读到这段代码都会
有一种膜拜之情油然而生，太牛逼了！

```c++
namespace ns {
    // a simple struct to model a person
    struct person {
        std::string name;
        std::string address;
        int age;
    };

    void to_json(json& j, const person& p) {
        j = json{{"name", p.name}, {"address", p.address}, {"age", p.age}};
    }

    void from_json(const json& j, person& p) {
        p.name = j.at("name").get<std::string>();
        p.address = j.at("address").get<std::string>();
        p.age = j.at("age").get<int>();
    }
}

ns::person p {"Ned Flanders", "744 Evergreen Terrace", 60};
json j = p;
ns::person copy = j;

assert(copy == p); // 当然，这里你需要实现 operator==
```

有了上面这些构造函数才使得下面这些语句合法合法。

```
json number = 1;
json str = "number";
```

上面这个构造函数基本上是通吃所有值类型的，但是有两个除外，`null` 和 `object`，
`basic_json`有一个专门处理`null`的构造函数如下：

```
basic_json(std::nullptr_t = nullptr) noexcept;
```

对于`object`则主要是修正问题，正常情况下，在下面这种情况：

```c++
json value = {
    {"key1", 1},
    {"key2", "2"},
};
```

会默认转换成一个`array`，因为 value 的初始化值其实是一个
`std::initializer_list<basic_json>`，其中每一个`element`都是一个`array`,但是对于
上面这种情况实际上应该解析成一个`object`才对。为了修正这个问题，`basic_json`还提
供了另一个构造函数：

```c++
basic_json(std::initializer_list<basic_json> init,
           bool type_deduction = true,
           value_t manual_type = value_t::array)
{
    // check if each element is an array with two elements whose first
    // element is a string
    bool is_an_object = std::all_of(init.begin(), init.end(),
                                    [](const basic_json & element)
    {
        return element.is_array() and element.size() == 2 and element[0].is_string();
    });

    // adjust type if type deduction is not wanted
    if (not type_deduction)
    {
        // if array is wanted, do not create an object though possible
        if (manual_type == value_t::array)
        {
            is_an_object = false;
        }

        // if object is wanted but impossible, throw an exception
        if (manual_type == value_t::object and not is_an_object)
        {
            JSON_THROW(type_error::create(301, "cannot create object from initializer list"));
        }
    }

    if (is_an_object)
    {
        // the initializer list is a list of pairs -> create object
        m_type = value_t::object;
        m_value = value_t::object;

        std::for_each(init.begin(), init.end(), [this](const basic_json & element)
        {
            m_value.object->emplace(*(element[0].m_value.string), element[1]);
        });
    }
    else
    {
        // the initializer list describes an array -> create array
        m_type = value_t::array;
        m_value.array = create<array_t>(init);
    }

    assert_invariant();
}
```

可以看出默认情况下，之前的代码会从 `array` 修正到 `object`。当然如果你的本意确实
是创建一个`array`的话可以使用静态成员函数：`json::array`

```
static basic_json array(std::initializer_list<basic_json> init =
                            std::initializer_list<basic_json>())
{
    return basic_json(init, false, value_t::array);
}
```

此外，为了接口的对称性，还存在一个静态方法`json::object`用于创建一个对象。

## `nlohmann::json` 到实际数据类型的自动转换

反方向的转换在[JSON for Modern C++][json]中同样存在，这个转换主要是通过下面这个
函数来实现的：

```c++
template < typename ValueType, typename std::enable_if < ...... , int >::type = 0 >
operator ValueType() const
{
    // delegate the call to get<>() const
    return get<ValueType>();
}
```

这个函数存在让下面这样的语法变得合法：

```c++
nlohmann::json json = {
    {"str", "hello"},
    {"int", 1},
    {"bool", true},
}

std::string sval = json["str"];
int ival = json["int"];
bool bval = json["bool"];
```

它实际上上的实现和 `to_json` 类似，不过使用的是`from_json`。

# 从另一个角度看待 `nlohmann::json` —— 容器

其实在设计上来说，`nlohmann::json` 被设计成了一个容器，你可以想象`std::map`，
`std::vector`拥有的那些API，`nlohmann::json`都存在。但是 `nlohmann::json` 除了表
示 `object` 和 `vector` 之外还表示 `number`，`true`，`false` 这些值。对于这些类
型的值来说，容器相关的那些 API 实际上没有真正的意义。从下面这张图可以看出这些相
关的 API 的具体实现情况：

![container](http://oqb1kuipb.bkt.clouddn.com/json-container-2017-5-14)

## 需要注意的地方

容器的 API 有 STL 基础的人基本上都非常熟悉，这里不再赘述，有几个和普通容器不对等
的地方需要大家注意：

1. `array` 的 index 如果超过大小 operator[]，不会出错，中间缺失的那些index会默认
   创建成空的 `json`对象。比如：

   ```
   json arr = json::array();

   arr[3] = "hello world";
   arr[5] = 42;

   std::cout << arr << std::endl;
   ```

   最终得到的输出是 `[null,null,null,"hello world",null,42]`

2. `array` 的 `operator[]` 只能用 `sizet_t` 调用（包括那些可以默认转换的），而
   `object` 的 `operator[]` 只能用 `std::string` 调用（也包括那些可以默认转换的
   ）。否则会出现运行时异常。这种错误目前似乎无法在编译期间检查出来。

## basic_json::value

`basic_json` 除了提供容器常用的接口 `operator[]` 和 `at` 之外，还提供了 `value`
成员函数用于取对象中的值，当值不存在的时候提供默认值。这个方法和 `Python` 中的
or 很像。

```python
retun x or "default"
```

```c++
json j = {
    {"exist", "hello"},
};

auto exist = j.value("exist", "");  // "hello"
auto noexist = j.value("noexist", "world");  // "world"
```

这个函数对于处理可选参数非常有用。

# 吐嘈

这个库非常强大，实现上也非常的精巧，但是有时候我会觉得，如果把 `JSON` 的值和
`JSON` 这两个概念分开表示可能结构上会清晰一些，让 `json` 单纯是个容器。当然这都
是感觉上的东西，实际上作者为什么没有这样做可能有他自己其他方面考虑。
