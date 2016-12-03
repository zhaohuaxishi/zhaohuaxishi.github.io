---
author: 郭荣飞
date: 2015-01-31
title: Netlink 库 -- 官方开发者教程中文版第五部分
tags:
 - netlink
 - libnl
---


<H1 id = "message_parse_construct">5. 消息的解析和构建</H1>

<H2 id = "msg_formate">5.1 消息格式</H2>

关于 `netlink` 协议以及它的消息格式请查看 [netlink 协议基础][nlp_fundaments] 一
节。

<H3>对齐</H3>

大部分的 `netlink` 协议对所有的边界都有严格的对齐方案。对齐值是由
`NLMSG_ALIGNTO` 定义的，这个值固定为 4 个字节。所以所有的 `netlink` 消息头部、有
效载荷段的开头、协议相关头部和属性段都必须从一个是 `NLMSG_ALIGNTO` 倍数的偏移量
开始。

<!--more-->

```cpp
	#include <netlink/msg.h>

	int nlmsg_size(int payloadlen);
	int nlmsg_total_size(int payloadlen);
```

这个库提供了一系列的函数来自动处理对齐问题。`nl_total_size()` 函数返回一个消息总
长度，这个长度包括了确保下一个消息头部已经正确对齐的填充（padding）长度。

	 <----------- nlmsg_total_size(len) ------------>
	 <----------- nlmsg_size(len) ------------>
	+-------------------+- - -+- - - - - - - - +- - -+-------------------+- - -
	|  struct nlmsghdr  | Pad |     Payload    | Pad |  struct nlsmghdr  |
	+-------------------+- - -+- - - - - - - - +- - -+-------------------+- - -
	 <---- NLMSG_HDRLEN -----> <- NLMSG_ALIGN(len) -> <---- NLMSG_HDRLEN ---

如果你想知道是否需要在一个消息的尾部进行填充处理，`nlmsg_padlen()` 函数会返回对
于一个给定的有效载荷长度需要进行填充的字节数目。

```cpp
	#include <netlink/msg.h>

	int nlmsg_padlen(int payloadlen);
```

<H2 id = "parse_msg">5.2 解析一个消息</H2>

这个库提供了两套完全不同的方法来解析 `netlink` 消息。它为那些需要手动完成所有解
析的应用程序提供了一套低层（low level）的 API。这种方式会在下面进行介绍。此外该
库也提供了另外一套接口用于把解析器实现成高速缓存（cache[^1]）操作的一部分，如果
你的协议是用来处理诸如网络链路、路由的任何类型的对象的时候这种方式将会非常有用。
这套高层（hight level）接口会在 [高速缓存系统][cache_system] 一节中介绍。

<H3>把字节流拆分成多条消息</H3>

通常从一个网络套接字中接收下来的是消息流。你将会得到一个缓冲区以及它的长度，这个
缓冲区可能包含任意数量的 `netlink` 消息。

第一个消息头部位于消息流的开始处。任何后续的消息都是通过在前一个头部的基础上调用
`nlmsg_next()` 函数来获得。

```cpp
	#include <netlink/msg.h>

	struct nlmsghdr *nlmsg_next(struct nlmsghdr *hdr, int *remaining);
```

`nlmsg_next()` 会自动从剩余字节数中减掉前一个消息的大小。

需要注意的是，在前一个消息中并没有指示是否存在下一个消息，你必须假设有更多的消息
存在直到消息流中的所有数据都被解析完成为止。

为了简化这个工作，`libnl` 库提供了另一个函数 `nlmsg_ok()`，如果消息流中的剩余字
节中还存在另一个消息的话，这个函数会返回 `true`。`nlmsg_valid_hdr()` 是一个类似
的函数，它检查的是一个给定的 `netlink` 消息中是否至少包含某个最小长度的有效载荷
。

```cpp
	#include <netlink/msg.h>

	int nlmsg_valid_hdr(const struct nlmsgdhr *hdr, int payload);
	int nlmsg_ok(const struct nlmsghdr *hdr, int remaining);
```

这些函数的一个典型使用方式如下：

```cpp
	#include <netlink/msg.h>

	void my_parse(void *stream, int length)
	{
		struct nlmsghdr *hdr = stream;

		while(nlmsg_ok(hdr, length)) {
			// Parse message here
			hdr = nlmsg_next(hdr, &length);
		}
	}
```

**注意：** nlmsg_ok() 只会在剩余缓冲区长度足够容纳包括消息有效载荷在内的整个消息
的时候才会返回 `true` 。如果只是包含部分消息的话它会返回 `false`

上面的例子可以使用 `nlmsg_for_each()` 迭代器改写：

```cpp
	#include <netlink/msg.h>

	struct nlmsghdr *hdr;

	nlmsg_for_each(hdr, stream, length) {
		/* 处理消息 */
	}
```

<H3>消息有效载荷</H3>

消息的有效载荷是附加在消息头部的末尾的，而且它一定是在 `NLMSG_ALIGNTO` 的倍数位
置开始。前面这点是通过在消息头部的末尾进行必要的填充来保证的。`nlmsg_data()` 函
数会根据消息计算好必要的偏移量然后返回消息有效负载开始处的指针。

```cpp
	#include <netlink/msg.h>

	void *nlmsg_data(const struct nlmsghdr *nlh);
	void *nlmsg_tail(const strcut nlmsghdr *nlh);
	int nlmsg_datalen(const struct nlmsgdhr *nlh);
```

消息有效负载的长度是通过 `nlmsg_datalen()` 返回的。

				       <--- nlmsg_datalen(nlh) --->
	    +-------------------+- - -+----------------------------+- - -+
	    |  struct nlmsghdr  | Pad |           Payload          | Pad |
	    +-------------------+- - -+----------------------------+- - -+
	nlmsg_data(nlh) ---------------^                                  ^
	nlmsg_tail(nlh) --------------------------------------------------^

有效负载可能包含任意的数据，但是可能会有严格的对齐以及格式标准，这一切都由实际的
`netlink` 协议决定。

<H3 id = "parse_msg_attr">消息属性</H3>

大部分的 `netlink` 协议都使用了 `netlink` 属性。它不但可以使得协议自身更容易理解
而且可以使得协议在日后扩展更加方便。新的协议可以在任意时刻添加而旧的属性也可以被
新的属性废弃掉而不会破坏协议的二进制兼容性。

				       <---------------------- payload ------------------------->
				       <----- hdrlen ---->       <- nlmsg_attrlen(nlh, hdrlen) ->
	    +-------------------+- - -+-----  ------------+- - -+--------------------------------+- - -+
	    |  struct nlmsghdr  | Pad |  Protocol Header  | Pad |           Attributes           | Pad |
	    +-------------------+- - -+-------------------+- - -+--------------------------------+- - -+
	nlmsg_attrdata(nlh, hdrlen) -----------------------------^

`nlmsg_attrdata()` 函数会返回一个属性段的指针。属性段的长度可以通过调用
`nlmsg_attrlen()` 函数得到。

	#include <netlink/msg.h>

	struct nlattr *nlmsg_attrdata(const struct nlmsghdr *hdr, int hdrlen);
	int nlmsg_attrlen(const struct nlmsgdhr *hdr, int hdrlen);

关于如何使用 `netlink` 的属性请查看 [属性][attr] 一节。

<H3>解析消息的便捷方式</H3>

`nlmsg_parse()` 函数一步完成一个完整的 `netlink` 消息的检验。如果 `hdrlen > 0`
的话，这个函数首先会调用 `nlmsg_valid_hdr()` 来检验这个消息中是否能容纳下协议头
部。如果还有有效载荷需要解析的话，该函数会假设这些有效载荷是属性然后按照属性来解
析有效载荷。当用于属性解析的时候，这个函数和 `nla_parse()` 函数是完全一样的，更
多详情请查看 [[core_attr_parse_easy]][core_attr_parse_easy]。

	int nlmsg_parse(struct nlmsghdr *hdr, int hdrlen, struct nlattr **attrs,
		int maxtype, struct nla_policy *policy);

在 [[core_attr_parse_easy]][core_attr_parse_easy] 中还有一个额外的例子，以及更加
详细的信息。

<H2 id="construct_msg">5.3. 构建消息</H2>

关于 `netlink` 消息格式以及对齐需求方面的信息请参考 [消息格式][msg_formate] 一节
。

消息的构建是基于 `struct nl_msg` 结构体的，它使用内部缓冲区来存储实际的
`netlink` 消息。`struct nlmsg` 结构体并不指向与 `netlink` 消息的头部。请使用
`nlmsg_hdr()` 来获取这条 `netlink` 消息的头部指针。

在分配结构体的时候，可以指定最大的消息长度。它的默认大小是一页（PAGE_SIZE）。构
造消息应用程序将会不断地从这个最大消息长度中为每个添加的头部和属性预留空间。这种
方式可以让构建一个跨越多个层代码的消息时下层不用知道上层需要的空间。

为什么需要设置最大消息长度呢？这个问题经常和被推荐的方法——使用 `realloc()` 函数
动态的重新分配消息有效载荷缓冲区——一起被提出来。虽然在构建消息时可以使用
`nlmsg_expand()` 函数重新分配缓冲区，但这会导致所有指向消息缓冲区的指针失效。这
会使得 `nlmsg_hdr()`、`nla_nest_start()` 和 `nla_nest_end()` 函数出错，所以这种
方式无法作为默认方式。

<H3>分配 <code>struct nl_msg</code> 结构体</H3>

构建一条新的 `netlink` 消息的第一步是分配一个 `struct nl_msg` 结构体来承载消息头
部和有效载荷。为了简化各种各样的操作，库中存在多个这类的函数。

```cpp
	#include <netlink/msg.h>

	struct nl_msg *nlmsg_alloc(void);
	void nlmsg_free(struct nl_msg *msg);
```

`nlmsg_alloc()` 函数是默认的分配函数，它使用默认的最大消息长度，也就是一页的大小
（`PAGE_SIZE`）来分配一条新的消息。应用程序可以通过调用
`nlmsg_set_defualt_size()` 函数来更改默认大小。

```cpp
	void 	nlmsg_set_defualt_size(size_t);
```

**注意： 调用 `nlmsg_set_default_size()`并不会改变已经分配了的消息的最大消息长度
。**


```cpp
	struct nl_msg *nlmsg_alloc_size(size_t max);
```

除了改变默认消息长度之外，你也可以使用 `nlmsg_alloc_size()` 函数使用一个给定的最
大消息长度来来分配一条消息。

如果 `netlink` 消息头部在分配的时候就已经确定，应用程序可以考虑使用[^2]
`nlmsg_inherit()` 函数。这个函数会使用默认的最大消息长度分配一条消息，然后把头部
拷贝到这条消息中去。以 `NULL` 为参数调用 `nlmsg_inherit()` 就等于是调用
`nlmsg_alloc()` 函数。

```cpp
	struct nl_msg * nlmsg_inherit(struct nlmsghdr *hdr);
```

除了上面的方式之外，你也可以调用 `nlmsg_alloc_simple()` ，这个函数接收消息类型和
消息标志为参数。它的作用和 `nlmsg_inherit()` 其实是一样的，只不过前者把两个常用
的头部字段作为参数而后者使用使用整个头部作为参数。

```cpp
	#include <netlink/msg.h>

	struct nl_msg *nlmsg_alloc_simple(int nlmsg_type, int flags);
```

<H3>附加 netlink 消息头部</H3>

分配了 `struct nl_msg` 结构体之后，你需要添加 `netlink` 消息头部，除非你使用了
`nlmsg_alloc_simple()` 或者 `nlmsg_inherit()` 函数。如果你使用了前面两个方法之后
还进行这一步操作的话，这一步操作给出的头部会覆盖前面两个函数中的头部。

```cpp
	#include <netlin/msg.h>

	struct nlmsghdr *nlmsg_put(struct nl_msg *msg, uint32_t port, uint32_t seqnr,
			int nlmsg_type, int payload, int nlmsg_flags);
```

`nl_put()` 函数会根据提供的 `nlmsg_type`、`nlmsg_flags`、`seqnr`、`port` 参数构
建一个消息头部，然后把数据拷贝到 `netlink` 消息中。`seqnr` 可以设置成
`NL_AUTO_SEQ`，这表示自动使用下一个可用的序列号。为了使用这一特性，这条消息必须
通过 `nl_send_auto()` 函数发送出去。和 `seqnr` 参数一样，`port` 参数可以被设置为
`NL_AUTO_PORT`，这表示使用套接字中的本地端口号作为源端口。这通常是一个不错误的主
意，除非你是在回复一个请求[^3]。关于如何填充消息头部的更多信息请参考 [Netlink 协
议基础][nlp_fundaments]
一节。

**注意： 应用程序可以使用 `payload` 参数在头部后面为额外的数据预留空间。如果
`payload` 的值大于 0 的话，这相当于是调用 `nlmsg_reserve(msg, payload,
NLMSG_ALIGNTO)`。关于为数据预留空间的更多信息请参考 「core_msg_reserve」**

示例:


```cpp
	#include <netlink/msg.h>



	struct nlmsghdr *hdr;

	struct nl_msg *msg;

	struct myhdr {

		uint32_t foo1, foo2;

	} hdr = { 10, 20 };



	/* 使用默认大小分配一条消息 */

	msg = nlmsg_alloc();


	/*
	 * 使用类型 MY_MSGTYPE, 标志 NLM_F_CREATE 构造消息头部，
	 * 让库来填充端口号和序列号并且为 struct myhdr 预留空间。
	 */

	hdr = nlmsg_put(msg, NL_AUTO_PORT, NL_AUTO_SEQ, MY_MSGTYPE, sizeof(hdr), NLM_F_CREATE);

	/* 把自己的消息头部拷贝到刚刚预留的空间 */

	memcpy(nlmsg_data(hdr), &hdr, sizeof(hdr));


	/*
	 * 消息的格式如下:
	 *     +-------------------+- - -+----------------+- - -+
	 *     |  struct nlmsghdr  | Pad |  struct myhdr  | Pad |
	 *     +-------------------+-----+----------------+- - -+
	 * nlh -^                        /                \
	 *                              +--------+---------+
	 *                              |  foo1  |  foo2   |
	 *                              +--------+---------+
	 */
```

<H3>在消息的末尾预留空间</H3>

虽然后面介绍的大部分函数都会为添加到 `neltink` 消息尾部的数据自动预留空间，但是
有些情况下，应用程序还是需要直接预留空间。

```cpp
	#include <netlink/msg.h>

	void *nlmsg_reserve(struct nl_msg *msg, size_t len, int pad);
```

`nlmsg_reserve()` 函数在 `netlink` 消息的默认预留 `len` 字节的空间，然后返回一个
指向这块预留空间开始位置的指针。`pad` 参数可以用来在预留空间之前把 `len` 对齐到
任意字节。

下面这个例子请求在消息末尾预留一块 17 字节的空间并对齐到 4 字节处，因此最后预留
的空间总共是 20 个字节。

```cpp
	#include <netlink/msg.h>

	void *buf = nlmsg_reserve(msg, 17, 4);
```

**注意： `nlmsg_reserve()` 不会对齐缓冲区的开头，任何对齐需求都必须由前面的消息
段的所有者来。**

<H3> 在消息的尾部添加数据</H3>

`nlmsg_append()` 函数在消息的末尾添加 `len` 字节的数据，并在请求了填充且需要填充
的情况下进行填充。

```cpp
	#include <netlink/msg.h>

	int nlmsg_append(strcut nl_msg *msg, void *data, size_t len, int pad);
```

这个方法等同于调用 `nlmsg_reserve()` 函数然后调用 `memcpy()` 函数把数据拷贝到刚
刚预留的空间中去。

**注意： `nlmsg_reserve()` 不会对齐缓冲区的开头，任何对齐需求都必须由前面的消息
段的所有者来。**

<H3>在消息中添加属性</H3>

关于如何构建属性和把属性添加到消息将会在 [属性][attr] 一节中介绍。

  [^1]: 这里的 `cache` 不是计算机组成原理里面提供的硬件，不要混淆。「译者注」
  [^2]: 原文中使用 `sue` 一词应该是 `use` 一词的笔误。「译者注」
  [^3]: 前面两句在是按照原文中的顺序翻译的，个人觉得顺序有颠倒。因为是使用自动序列号在应答消息中不适用（应答消息的序列号可能和请求一致）。「译者注」

  [nlp_fundaments]: /netlink/2015/01/23/libnl-translation-part2.html#nlp_fundamentals
  [cache_system]: /netlink/2015/02/15/libnl-translation-part8.html#cache_system
  [attr]: /netlink/2015/02/12/libnl-translation-part6.html#attributes
  [core_attr_parse_easy]: /netlink/2015/02/12/libnl-translation-part6.html#core_attr_parse_easy

  [msg_formate]: /netlink/2015/01/23/libnl-translation-part2.html#message_formate
