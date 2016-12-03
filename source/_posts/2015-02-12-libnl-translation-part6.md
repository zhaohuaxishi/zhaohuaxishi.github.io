---
author: 郭荣飞
date: 2015-02-12
title: Netlink 库 -- 官方开发者教程中文版第六部分
tags:
 - netlink
 - libnl
---

<H1 id = "attributes">6. 属性</H1>

如果可能的话，无论什么时候你都应该把 `netlink` 消息的有效载荷编码成 `netlink` 属
性。使用属性可以使得日后对任何 `netlink` 协议进行扩展的时候都不会破坏它的二进制
兼容性。比如：假设你的设备现在可能使用一个 32 位的计数器来统计数据，但是几年后设
备改成维护一个 64 位的计数器来记录更快的网络硬件。如果你的协议使用了属性，那么过
度到 64 位计数器是一个非常简单的事情，你只是需要发送一个新的属性来包含这个 64 位
的变量，与此同时仍然提供旧的 32 位的计数器。如果你的协议没有使用属性的话，你将无
法在不祸及已有的协议使用者的前提下转化这个数据类型。

属性嵌套的概念也允许你的协议的子系统实现和维护自己的属性模式。假如现在引入了新一
代的网络设备，而它需要设置一系列的协议设计之初没有考虑到的新的配置选项。通过使用
属性，新一代的设备可以定义一个新的属性然后用自己的子属性结构来填充这个属性，这些
子属性可以扩展甚至是完全废弃原来的属性。

所以，请*始终*使用属性，即便是在你几乎可以肯定你的消息格式永远都不会改变的情况下
也是如此。

<!--more-->

<H2 id = "attr_formate">6.1 属性格式</H2>

`netlink` 属性允许你往 `neltink` 消息中添加任意数量的任意长度数据段。关于属性在
消息中的存放位置请参考 [【core_msg_attr】][core_msg_attr] 一节。

`nlmsg_attrdata()` 函数返回的属性数据的格式如下：

	 <----------- nla_total_size(payload) ----------->
	 <---------- nla_size(payload) ----------->
	+-----------------+- - -+- - - - - - - - - +- - -+-----------------+- - -
	|  struct nlattr  | Pad |     Payload      | Pad |  struct nlattr  |
	+-----------------+- - -+- - - - - - - - - +- - -+-----------------+- - -
	 <---- NLA_HDRLEN -----> <--- NLA_ALIGN(len) ---> <---- NLA_HDRLEN ---

每一个属性都应该在 `NLA_ALIGNTO` （4 字节）倍数的偏移量上开始。如果你想知道属性
的末尾是否需要进行填充，你可以使用 `nla_padlen()` 函数，它返回需要添加的填充的字
节数。

 ![属性格式][attr_formate_img]

每一个属性都是和一个类型及长度字段一起编码，这两个字节都是 16 个字节，它们存储在
属性的有效载荷前面的属性头部中（struct nlattr）。属性的长度是用来计算下一个属性
的偏移量的。

<H2 id = "parse_attr">6.2 属性解析</H2>

<H3 id = "core_attr_parse_split">把属性流分割成属性集</H3>

库中存在一个手动分割属性流的接口，虽然大部分的应用程序都会使用 `nlmsg_parse()`
家族的函数之一（参考 [core_attr_parse_easy][attr_parse_easy]）来完成这项工作。

在[【属性格式】][attr_formate] 一节中提到，属性段中包含了不定数量的属性序列。
`nlmsg_attrdata()` 函数返回的指针指向第一个属性的头部。任何后续的属性都可以通过
在前一个属性头部上调用 `nla_next()` 来获得。

```cpp
	#include <netlink/attr.h>

	struct nlattr *nla_next(const struct nlattr *attr, int *remaining);
```

这个函数的使用场景和 `nlmsg_next()` 函数是一样的，所以 `nla_next()` 函数也会从属
性流的剩余字节数中减去前一个属性的大小。

和消息一样，属性中并没有包含接下来是否还有其他属性的指示器。唯一可以用来判别的是
属性流的剩余字节数。`nla_ok()` 函数可以用来判断剩余字节数中是否还存在其他的属性
。

```cpp
	#include <netlink/attr.h>

	int nla_ok(const struct nlattr *attr, int remaining);
```

`nla_ok()` 和 `nla_next()` 函数的惯常用法如下：

**nla_ok()/nla_next()用法**

```cpp
	#include <netlink/msg.h>
	#include <netlink/attr.h>

	struct nlattr *hdr = nlmsg_attrdata(msg, 0);
	int remaining = nlmsg_attlen(msg, 0);

	while(nla_ok(hdr, remaining)) {
		/* 在此解析属性 */

		hdr = nla_next(hdr, &remaining);
	}
```

**注意：**`nla_ok()` 只有在剩余字节数中包含包括属性有效载荷在内的完整属性的时候
才会返回 `true`

<H3>访问属性的头部和属性的有效载荷</H3>

无论是使用分割属性流的方式还是还是使用其他接口，只要找出来一个属性你就可以访问它
的载荷和头部。

				     <- nla_len(hdr) ->
	    +-----------------+- - -+- - - - - - - - - +- - -+
	    |  struct nlattr  | Pad |     Payload      | Pad |
	    +-----------------+- - -+- - - - - - - - - +- - -+
	nla_data(hdr) ---------------^

`nla_len()` 函数和 `nla_type()` 函数可以用来访问属性的头部。`nla_len()` 会返回不
包括填充字节在内的载荷的长度。`nla_type()` 函数则会返回属性的类型。

```cpp
	#include <netlink/attr.h>

	int nla_len(const struct nlattr *hdr);
	int nla_type(const struct nlattr *hdr);
```

`nla_data()` 函数会返回指向属性有效载荷的指针。需要注意的是因为 `NLA_ALIGENTO`
是 4 个字节所以直接转型或解引用任何大于 32 位的数据类型的指针都是不安全的做法，
这依赖于应用程序所运行的平台。

```cpp
	#include <netlink/attr.h>

	void *nla_data(struct nlattr *hdr);
```

**注意：** 千万不要断定载荷的大小是你期望的大小，*务必*自己检查载荷的长度的来确
认它满足你的要求

<H3 id = "core_attr_validation">属性检验</H3>

当接收到一个 `netlink` 属性的时候，接收者对属性应该是什么样子有自己的期望。这种
期望必须被定义出来以保证发送方满足我们的期望。出于这个目的，库中提供了一个属性验
证的接口，它必须在访问任何的载荷之前使用。

任何提供属性验证的函数都是基于 `struct nla_policy` 结构体的：

```cpp
	struct nla_policy {
		uint16_t	type;
		uint16_t 	minlen;
		uint16_t 	maxlen;
	};
```

`type` 字段指定了属性的数据类型，比如：`NLA_U32`、`NLA_STRING`、`NLA_FLAG`，它的
默认值是 `NLA_UNSPEC`。`minlen` 字段定义了合法属性的最小的有效载荷长度。`minlen`
的值对于大部分诸如整型、标志这类的基本数据类型来说都是隐含的。`maxlen` 字段则定
义了合法属性的最大有效载荷长度。

**注意：** 当我们把结构体编码到一个属性中去的时候，指定最大的有效载荷长度并非明
智之举因为这不利于日后扩展该结构体。而在 `neltink` 协议中一项常见任务就是扩展协
议而不会破坏它的向后兼容性

`nla_validate()` 是使用 `struct nla_policy` 结构体的函数之一。这个函数接收一个
`struct nla_policy` 数组并使用属性类型作为下标访问数组。如果一个属性类型越界的话
这个属性会被当做是合法的属性。这个为了使旧的应用程序在无法识别新引入的属性的情况
下能正常工作而特意进行的处理。

```cpp
	#include <netlink/attr.h>

	int nla_validate(struct nlattr *head, int len, int maxtype, struct nla_policy *policy)
```

如果属性合法 `nla_policy()` 函数返回 0，否则它返回一个和错误相关的错误码。

大部分应用程序不会直接使用 `nla_validate()` 而是使用 `nla_parse()` 函数。这个函
数以同样的方式验证属性，但是在验证的同时它还会解析属性。
[core_attr_parse_easy][attr_parse_eray] 一节中有更多的信息以及一个例子程序。

验证属性的详细步骤如下：

    1. 如果属性类型是 0 或者大于最大的属性值，返回 0。

    2. 如果属性的有效载荷长度小于 minlen，返回 -NLE_ERANGE。

    3. 如果定义了 maxlen 而有效载荷长度超过了这个值， 返回 -NLE_ERANGE。

    4. 检验数据类型相关的规则，参考属性数据类型一节。

    5. 如果上面的全部条件都满足的话返回 0。

<H3 id = "core_attr_parse_easy">解析属性的便捷方式</H3>

大部分的应用程序不会想要按照[core_attr_parse_split][] 中提到的方式自己分割属性流
。一种更加便捷的方式是使用 `nla_parse()` 函数。

	#include <netlink/attr.h>

	int nla_parse(struct nlattr **attrs, int maxtype, struct nlattr *head,
		int len, struct nla_policy *policy);

`nla_parse()` 函数会迭代一条属性流，按照 [core_attr_validation][attr_validation]
中提到的方式验证每一个属性，如果所有的属性都通过了验证的话，指向每一个属性的指针
会被存放在属性数组的 `attrs[nla_type(attr)]` 位置中。

除了 `nla_parse()` 函数之外你也可以使用 `nlmsg_parse()` 函数把解析消息和解析消息
的属性的工作在一个步骤中完成。关于如何使用这些函数请参考
[core_attr_parse_easy][attr_parse_easy] 一节。

<H3>例子</H3>

下面的例子展示了如何解析一个通过 `neltink` 协议发送的不包含协议头部的 `neltink`
消息。但是这个例子强制使用了属性规则（policy），`MY_ATTR_FOO` 必须是 32 位的整数
而 `MY_ATTR_BAR` 必须是一个最多包含 16 个字符的字符串。

	#include <netlink/msg.h>
	#include <netlink/attr.h>

	enum {
		MY_ATTR_FOO = 1,
		MY_ATTR_BAR,
		__MY_ATTR_MAX,
	};


	#define MY_ATTR_MAX (__MY_ATTR_MAX - 1)


	static struct nla_policy my_policy[MY_ATTR_MAX+1] = {
		[MY_ATTR_FOO] = { .type = NLA_U32 },
		[MY_ATTR_BAR] = { .type = NLA_STRING,
				  .maxlen = 16 },
	};


	void parse_msg(struct nlmsghdr *nlh)
	{
		struct nlattr *attrs[MY_ATTR_MAX+1];

		if (nlmsg_parse(nlh, 0, attrs, MY_ATTR_MAX, my_policy) < 0)
			/* 出错处理 */

		if (attrs[MY_ATTR_FOO]) {
			/* 消息中存在 MY_ATTR_FOO 类型的属性 */
			printf("value: %u\n", nla_get_u32(attrs[MY_ATTR_FOO]));
		}
	}

<H3>定位单个属性</H3>

如果一个应用程序只在乎单个属性，那么它可以使用 `nla_find()` 函数和
`nlmsg_find_attr()` 函数。这些函数会迭代所有的属性，找到一个符合条件的属性然后发
挥这个属性头部的指针。

	#include <netlink/attr.h>

	struct nlattr *nla_find(struct nlattr *head, int len, int attrtype);




	#include <netlink/msg.h>

	struct nlattr *nlmsg_find_attr(struct nlmsghdr * hdr, int hdrlen, int attrtype);

**注意：** `nla_find()` 和 `nlmsg_find_attr()` 不会搜索嵌套的属性，参考
[嵌套属性][nested_attr] 一节


<H3>迭代属性流</H3>

有些时候为属性流中的每一个属性都设置一个唯一的属性类型并不太合理。比如一个列表可
以通过一系列的属性来传输，即便每一个属性的属性类型是都是递增的，使用
`nlmsg_parse()` 或者 `nla_parse()` 函数填充一个数组的做法也是不合理的[^1]。

因此库中提供了迭代属性流的方法：

    #include <netlink/attr.h>

	nla_for_each_attr(attr, head, len, remaining)

`nla_for_each_attr()` 是一个宏，你可以在代码块之前使用：

	#include <netlink/attr.h>

	struct nlattr *nla;
	int rem;

	nla_for_each_attr(nla, attrstream, streamlen, rem) {
		/* 验证并解析属性 */

	}

	if (rem > 0)
		/* 没有解析的数据 */

<H2 id = "attr_construct">6.3. 属性的构造</H2>

往 `neltink` 消息中添加属性的接口是建立在普通的消息构建接口之上的。这里假设消息
的头部和实际的 `netlink` 协议的头部已经添加到消息中了。

	struct nlattr * nla_reserve(struct nl_msg *msg, int attrtype, int len);

`nla_reserve()` 函数在消息的末尾添加一个属性头部并且为属性载荷预留 `len` 字节的
空间。这个函数会返回消息中的属性载荷段的指针。为了保证下一个属性正确的对齐，这个
函数还会在添加的属性的末尾进行必要填充。

	int nla_put(struct nl_msg *msg, int attrtype, int attrlen, const void *data);

`nla_put()` 函数以 `nla_reserve()` 函数为基础，只不过它还接收一个指向包含属性载荷
的缓冲区的指针。这个函数会自动把数据从缓冲区拷贝到消息中去。

**例子**

	struct my_attr_sturct {
		uint32_t a;
		uint32_t b;
	};

	int my_put(struct nl_msg *msg)
	{
		struct my_attr_sturct obj = {
			.a = 10,
			.b = 20,
		};

		return nla_put(msg, ATTR_MY_STRUCT, sizeof(obj), &obj);
	}

关于数据类型相关的属性构造请查看 [属性数据类型][attr_datatype] 一节。

<H2>基于异常的属性构造</H2>

和内核 API 类似，这个库提供了基于异常的接口。这些宏的行为和他们对应的方法是一致的，
不同之处在出错时这些宏会跳转到 `nla_put_failure` 位置。

**例子**[^2]

	#inlcude <netlink/msg.h>
	#include <netlink/attr.h>

	void construct_attrs(struct nl_msg *msg)
	{
		NLA_PUT_STRING(msg, MY_ATTR_FOO1, "some text");
		NLA_PUT_U32(msg, MY_ATTR_FOO2, 0x1010);
		NLA_PUT_FLAG(msg, MY_ATTR_FOO3, 1);

		return 0;

	nla_put_failure:
		/* NLA_PUT 系列的宏会在出错的时候跳转到这个位置 */
		return EMSGSIZE;
	}

关于数据类型相关的异常处理变种请查看 [属性数据类型][attr_datatype] 一节。

<H2 id = "attr_datatype">6.4. 属性数据类型</H2>

为了简化属性的访问和验证库中定义了许多基本数据类型。属性的数据类型并没有编码在
属性中，所以发送方和接收方对于哪一个属性属于何种数据类型要有一致的定义。

<table rules="all" width="100%" frame="border" cellspacing="0" cellpadding="4">
	<col width="16%" />
	<col width="83%" />
	<thead>
		<tr>
			<th align="left" valign="top"> 类型             </th>
			<th align="left" valign="top"> 描述</th>
		</tr>
	</thead>

	<tbody>
		<tr>
			<td align="left" valign="top"><p class="table"><code>NLA_UNSPEC</code></p></td>
			<td align="left" valign="top"><p class="table">不确定的属性</p></td>
		</tr>
		<tr>
			<td align="left" valign="top"><p class="table"><code>NLA_U{8|16|32}</code></p></td>
			<td align="left" valign="top"><p class="table">整型</p></td>
		</tr>
		<tr>
			<td align="left" valign="top"><p class="table"><code>NLA_STRING</code></p></td>
			<td align="left" valign="top"><p class="table">字符串</p></td>
		</tr>
		<tr>
			<td align="left" valign="top"><p class="table"><code>NLA_FLAG</code></p></td>
			<td align="left" valign="top"><p class="table">标志</p></td>
		</tr>
		<tr>
			<td align="left" valign="top"><p class="table"><code>NLA_NESTED</code></p></td>
			<td align="left" valign="top"><p class="table">嵌套属性</p></td>
		</tr>
	</tbody>
</table>


这些数据类型除了能够简化属性访问之外，最大的优势在于对于每个属性会基于一个 `policy`
自动进行合法性检验。这种检验会通过检查载荷的最小长度（对某些数据类型也会进行载荷
最大长度的检验）来保证访问载荷的安全性。

<H3>6.4.1. 整型属性</H3>

使用得最多的数据类型是整型。整型有四种不同的大小：

	NLA_U8	 	8  位的整数
	NLA_U16 	16 位的整数
	NLA_U32 	32 位的整数
	NLA_U64 	64 位的整数

注意由于对齐需求的存在，使用 `NLA_U8` 和 `NLA_U16` 规格的整型起不到节省 `nelink`
消息空间的作用，使用它们只是为了限制属性值的范围。

**解析整型属性**

	#include <neltink/attr.h>

	uint8_t nla_get_u8(struct nlattr *hdr);
	uint16_t nla_get_u16(struct nlattr *hdr);
	uint32_t nla_get_u32(struct nlattr *hdr);
	uint64_t nla_get_u64(struct nlattr *hdr);

例子：

	if (attrs[MY_ATTR_FOO])
		uint32_t val = nla_get_u32(attrs[MY_ATTR_FOO]);

**构建整型属性**

	#include <neltink/attr.h>

	int nla_put_u8(struct nl_msg *msg, int attrtype, uint8_t value);
	int nla_put_u16(struct nl_msg *msg, int attrtype, uint16_t value);
	int nla_put_u32(struct nl_msg *msg, int attrtype, uint32_t value);
	int nla_put_u64(struct nl_msg *msg, int attrtype, uint64_t value);

基于异常的对应函数：

	NLA_PUT_U8(msg, attrtype, value);
	NLA_PUT_U16(msg, attrtype, value);
	NLA_PUT_U32(msg, attrtype, value);
	NLA_PUT_U64(msg, attrtype, value);

**验证**

如果你在在填充一个 `struct nl_policy` 数组的时候使用 `NLA_U8`、`NLA_U16`、`NLA_U32`
或者 `NLA_U64` 来定义整数类型，`libnl` 库会保证这些 `policy` 使用正确的的最小载荷
长度。

验证并不区分有符号和无符号整数，唯一的决定因素是数据的大小。如果应用程序需要保证
数据在某个特定的范围的话，它需要自己完成这一项检验。

	static struct nla_policy my_policy[ATTR_MAX+1]={
		[ATTR_FOO]={.type=NLA_U32},
		[ATTR_BAR]={.type=NLA_U8},
	};

上面这段代码等价于：

	static struct nla_policy my_policy[ATTR_MAX+1]={
		[ATTR_FOO]={.minlen=sizeof(uint32_t)},
		[ATTR_BAR]={.minlen=sizeof(uint8_t)},
	};

<H3>6.4.2. 字符串属性</H3>

字符串类型用来标识一个以 `NUL` 结尾的不定长字符序列。它不是用来标识二进制数据流。

字符串属性的有效载荷可以通过调用 `nla_get_string()` 函数来获得。`nla_strdup()`
函数会对属性载荷调用 `strdup()` 函数并返回新分配的字符串。

	#inlcude <neltink/attr.h>

	char *nla_get_string(struct nlattr *hdr);
	char *nla_strdup(struct nlattr *hdr);

字符串属性是通过 `nla_put_string()` 或者对应的 `NLA_PUT_STRING()` 宏分配的。属性
的长度是 `strlen() + 1`，因为末尾的 `NUL` 字符也会包括在内。

	int nla_put_string(struct msg, int attrtype, const char *data);
	NLA_PUT_STRING(msg, attrtype, data);

为了验证字符串属性，你可以在定义 `struct nla_policy` 时使用 `NLA_STRING` 类型。
这个类型意味着载荷的最短长度是 1 字节，并且会检查 `NUL` 字符的存在。此外你可以
使用 `maxlen` 字段来定义字符序列的最大长度（包括末尾的 `NUL` 字符）。

	static struct nla_policy my_policy[] = {
		[ATTR_FOO] = {
			.type = NLA_STRING,
			.maxlen = IFNAMSIZ,
		},
	};

<H3>6.4.3 标志属性</H3>

标志属性代表一个布尔数据类型。如果属性存在则表示值为真，不存在则表示值为假。所以
标志属性的载荷长度永远都是 `0`。

	int nla_get_flag(struct nlattr *hdr);
	int nla_put_flag(struct nl_msg, int attrtype);

为了方便验证，库中也定义了 `NLA_FLAG` 属性。它表示 `maxlen` 为 `0` 也就保证了最大
的载荷长度是 `0`。

**例子**

	/* nla_put_flag() 在消息中添加一个长度为 0 的属性。*/
	nla_put_flag(msg, ATTR_FOO);

	/* 没有对应的接收方法，因为属性的存在与否代表了属性的值 */
	if (attrs[ATTR_FOO])
		/* 标志存在 */

<H3 id = "nested_attr">6.4.4 嵌套属性</H3>

在 [属性][attrs] 一节中已经提到，属性可以嵌套以表达属性之间的复杂树形结构。属性
嵌套也通常也用来表示一个子系统在消息中的子段。嵌套属性通常还会用来传输对象的列表。

当进行属性嵌套的时候，嵌套的属性作为父属性（容器属性-container）的有效载荷而存在。

**注意：** 当使用 `nlmsg_validate()`、`nlmsg_parse()`、`nla_validate()` 或者
`nla_parse()` 函数来验证属性的时候，它们只会验证第一层的属性。这些函数都不会递归
的验证属性。所以你必须对每一层的嵌套属性显式的调用 `nla_validate()` 或者
`nla_parse_nested()`

当定义一个嵌套属性的 `struct policy` 结构的时候你应该使用 `NLA_NESTED` 类型。它
不会强制使用任何最小载荷长度，除非你自己提供了这个字段。这是因为某些 `neltink`
协议允许空的容器属性。

	static struct nla_policy my_policy[] = {
		[ATTR_FOO] = {.type = NLA_NESTED},
	};

**嵌套属性的解析**

`nla_parse_nested()` 函数可以用来解析嵌套属性。它的行为和 `nla_parse()` 函数是
一致的，只不过它接收一个 `struct nlattr` 作为参数并且会使用这个属性的有效载荷作为
属性流进行解析。

	if (attrs[ATTR_OPTS]) {
		struct nlattr *nested[NESTED_MAX+1];
		struct nla_policy nested_policy[] = {
			[NESTED_FOO] = {.type = NLA_U32},
		};

		if (nla_parse_nested(nested, NESTED_MAX, attrs[ATTR_OPTS], nested_policy) < 0)
			/* 错误*/

		if (nested[NESTED_FOO])
			uint32_t val = nla_get_u32(nested[NESTED_FOO]);
	}

**嵌套属性的构造**

属性的嵌套是通过在代码前后分别调用 `nla_nest_start()` 和 `nla_nest_end()` 来完成的。
`nla_nest_start()` 函数会在消息中添加一个没有实际载荷的属性头部，在此之后添加的
数据都会成为容器属性的载荷部分直到调用 `nla_nest_end()` 为止，它的调用“关闭”了
容器属性并校正它的载荷长度以包含所有的数据长度。

	int put_opts(struct nl_msg *msg)
	{
		struct nlattr *opts;

		if (!(opts = nla_nest_start(msg, ATTR_OPTS)))
			goto nla_put_failure;

		NLA_PUT_U32(msg, NESTED_FOO, 123);
		NLA_PUT_STRING(msg, NESTED_BAR, "some text");

		nla_nest_end(msg, opts);
		return 0;

	nla_put_failure:
		nla_nest_cancle(msg, opts);
		return -EMSGSIZE;
	}

<H3>6.4.5 不定属性</H3>

这是默认的属性类型，它在没有合适的基本数据类型的时候使用。它用来表示任意长度任意
类型数据。

库中存在一个特殊的接口允许基于 `netlink` 属性分配抽象的地址对象，这个对象包含某种
形式的网络地址。请参考 [地址分配][addr_alloc] 一节以便获取更多的信息。

关于如何基于 `neltink` 属性分配抽象数据对象的更多信息请查看
[抽象数据分配][abstract_data_alloc] 一节。

这种类型的属性可以使用 `nla_get()` 和 `nla_put()` 函数来获取和构造属性。请查看
[属性构造][attr_construct] 一节中的例子。

<H2 id = "attr_examples">6.5. 示例程序</H2>

<H3>6.5.1. 使用属性构造 <code>netlink</code> 消息</H3>

	struct nl_msg *build_msg(int ifindex, struct nl_addr *lladdr, int mtu)
	{
		struct nl_msg *msg;
		struct nlattr *info, *vlan;
		struct ifinfomsg ifi = {
			.ifi_family = AF_INET,
			.ifi_index = ifindex,
		};

		/* 使用默认大小分配一条消息 */
		if (!(msg = nlmsg_alloc_simple(RTM_SETLINK, 0)))
			return NULL;

		/* 添加协议相关的头部 (struct ifinfomsg)*/
		if (nlmsg_append(msg, &ifi, sizeof(ifi), NLMSG_ALIGNTO) < 0)
			goto nla_put_failure

		/* 使用一个 32 位整型的属性来承载 MTU */
		NLA_PUT_U32(msg, IFLA_MTU, mtu);

		/* 使用一个不定类型属性来承载链路层地址 */
		NLA_PUT_ADDR(msg, IFLA_ADDRESS, lladdr);

		/* 添加一个容器属性以便使用嵌套属性来承载链路信息 */
		if (!(info = nla_nest_start(msg, IFLA_LINKINFO)))
			goto nla_put_failure;

		/* 往容器属性中添加一个字符串属性 */
		NLA_PUT_STRING(msg, IFLA_INFO_KIND, "vlan");

		/*
		 * 在打开的容器属性中再添加一个容器用来携带
		 * vlan 相关的属性
		 */
		if (!(vlan = nla_nest_start(msg, IFLA_INFO_DATA)))
			goto nla_put_failure;

		/* 在此添加和 vlan 相关的信息... */

		/* 完成 vlan 相关信息的填充并关闭第二个容器. */
		nla_nest_end(msg, vlan);

		/* 完成 link 信息属性的嵌套并关闭第一个容器. */
		nla_nest_end(msg, info);

		return msg;

	nla_put_failure:
		nlmsg_free(msg);
		return NULL;
	}

<H3>6.5.2 解析包含属性的 <code>netlink</code> 消息</H3>

	int parse_message(struct nlmsghdr *hdr)
	{
		/*
		 * policy 定义了两个属性: 一个 32 位整数和一个用来进行属性
		 * 嵌套的容器属性。
		 */
		struct nla_policy attr_policy[] = {
			[ATTR_FOO] = { .type = NLA_U32 },
			[ATTR_BAR] = { .type = NLA_NESTED },
		};
		struct nlattr *attrs[ATTR_MAX+1];
		int err;

		/*
		 * nlmsg_parse() 函数会保证消息的有效载荷至少足够容纳协议头部
		 * (struct my_hdr), 验证消息中包含的所有属性然后把属性的头部指针
		 * 按照属性类型存储在 attrs[] 数组中。
		 */
		if ((err = nlmsg_parse(hdr, sizeof(struct my_hdr), attrs, ATTR_MAX,
				       attr_policy)) < 0)
			goto errout;

		if (attrs[ATTR_FOO]) {
			/*
			 * 你现在可以安全的直接访问属性载荷而不用进行任何的检查
			 * 因为 nlmsg_parse() 函数会保证 policy 得到满足
			 */
			uint32_t foo = nla_get_u32(attrs[ATTR_FOO]);
		}

		if (attrs[ATTR_BAR]) {
			struct *nested[NESTED_MAX+1];

			/*
			 * 嵌套在容器中的属性可以和顶层属性一样解析。
			 */
			err = nla_parse_nested(nested, NESTED_MAX, attrs[ATTR_BAR],
					       nested_policy);
			if (err < 0)
				goto errout;

			// 在此处理嵌套属性。
		}

		err = 0;
	errout:
		return err;
	}

   [^1]: 文中并没有说明不合理的理由，我的理解是列表数的数量是不固定的，不适合使用数组。【译者注】
   [^2]: 原文中的例子里面 `MY_ATTR_FOO2` 写成了 `MY_ATTR_FOO1` 个人认为是笔误。【译者注】

   [core_msg_attr]: /netlink/2015/01/31/libnl-translation-part5.html#parse_msg_attr
   [attr_formate_img]: http://www.infradead.org/~tgr/libnl/doc/images//attribute_hdr.png
   [attr_validation]: #core_attr_validation
   [attr_parse_easy]: #core_attr_parse_easy
   [attr_formate]: #attr_formate
   [attr_construct]: #attr_construct
   [core_attr_parse_split]: #core_attr_parse_split
   [attr_datatype]: #attr_datatype
   [attrs]: #attributes
   [nested_attr]: #nested_attr
   [addr_alloc]: /netlink/2015/02/15/libnl-translation-part9.html#abstract_addr
   [abstract_data_alloc]: /netlink/2015/02/15/libnl-translation-part9.html#abstract_data

