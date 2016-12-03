---
author: 郭荣飞
date: 2015-02-17
title: Netlink 库 -- 官方开发者教程中文版第九部分
tags:
 - netlink
 - libnl
---

<H1 id = "abstract_data_type">9. 抽象数据类型</H1>

`libnl` 的核心库中实现了一些大多数 `neltink` 协议都会用到的高层的抽象数据类型。
如果需要的话，未来可能还会实现更多。

<!--more-->

<H2 id = "abstract_addr">9.1 抽象地址</H2>

大部分的 `neltink` 协议都是处理网络相关问题的，所以处理网络地址是一项常见的工作。

目前已经支持下面这些网络地址簇：

  - AF_INET
  - AF_INET6
  - AF_LLC
  - AF_DECnet
  - AF_UNSPEC

**地址分配**

`nl_addr_alloc()` 分配一个新的空地址。`maxsize` 参数定义了一个网络地址的最长
字节数。一个地址的大小和地址簇相关。如果一个地址簇和地址数据在分配的时候就已经
知晓的话，你可以选择使用 `nl_addr_build()` 函数。你也可以通过调用 `nl_addr_clone()`
函数来克隆一个地址。

	#include <netlink/addr.h>

	struct nl_addr *nl_addr_alloc(size_t maxsize);
	struct nl_addr *nl_addr_clone(struct nl_addr *addr);
	struct nl_addr *nl_addr_build(int family, void *addr, size_t size);

如果地址是通过 `netlink` 属性传递的，你可以通过 `nl_addr_alloc_attr()` 函数根据
提供的属性的载荷分配一个新的地址。`family` 参数是用来指定一个地址簇的，如果你
不知道的话请把它设置成 `AF_UNSPEC`。

	#include <netlink/addr.h>

	struct nl_addr *nl_addr_alloc_attr(struct nlattr *attr, int family);

如果地址是由用户提供的，它通常是存储在一个易于人类阅读的格式中的。`nl_addr_parse()`
可以用来解析一个代表地址的字符序列然后基于它分配一个新的地址。

	#include <netlink/addr.h>

	int nl_addr_parse(const char *addr, int hint, struct nl_addr **result);

如果分配成功的话，这个函数会返回 `0` 并新分配的地址存放在 `*result` 中。

**注意：** 记得在使用完地址之后使用 `nl_addr_put()` 函数解除对它的引用以释放空间。

**例子：把字符串转换成抽象地址**

	struct nl_addr *a = nl_addr_parse("::1", AF_UNSPEC);
	printf("Address family: %s\n", nl_af2str(nl_addr_get_family(a)));
	nla_addr_put(a);

	a = nl_addr_parse("11:22:33:44:55:66", AF_UNSPEC);
	printf("Address family: %s\n", nl_af2str(nl_addr_get_family(a)));
	nl_addr_put(a);

**地址引用计数**

抽象地址使用引用计数器来统计某个地址的使用总数。当最后一个用户解除对地址的引用
之后这个地址会被释放掉。

如果你把一个地址对象传递给另一个函数而你不知道这个函数将要使用这个地址多长时间，
这个时候确保你调用 `nl_addr_get()` 来活取一个额外的引用，然后让那个函数或者是代码
执行路径在用完地址之后尽可能早的调用 `nl_addr_put()`。

	#include <netlink/addr.h>

	struct nl_addr *nl_addr_get(struct nl_addr *addr);
	void nl_addr_put(struct nl_addr *addr);
	int nl_addr_shared(struct nl_addr *addr);

你可以在任何时候通过调用 `nl_addr_shared()` 函数来检查自己是不是这个地址的唯一
使用者。

**地址属性**

地址簇通常是在分配的时候就已经设置好了的。但是如果那时这个地址还不确定的话，你
可以在之后调用 `nl_addr_set_family()` 函数来设置它，同样你可以调用
`nl_addr_get_family()` 函数来取得地址信息。

	#include <netlink/addr.h>

	void nl_addr_set_family(struct nl_addr *addr, int family);
	int nl_addr_get_family(struct nl_addr *addr);

这一点对于地址的实际数据同样是成立的，它通常是在分配的时候就给出了。但是如果没有
在那时指定的话，你可以之后通过调用 `nl_addr_set_binary_addr()` 来设置或者是覆盖它。
需要注意的是新设置的值的长度不能超过地址分配的时候给出的最大值。地址数据是通过
`nl_addr_get_binary_addr()` 函数取得而它的长度则可以通过调用 `nl_addr_get_len()`
获得。

	#include <netlink/addr.h>

	int nl_addr_set_binary_addr(struct nl_addr, void *data, size_t size);
	void *nl_addr_get_binary_addr(struct nl_addr *addr);
	unsigned int nl_addr_get_len(struct nl_addr *addr);

如果你只想检验地址数据是否全是由零组成的。你可以调用 `nl_addr_iszero()` 函数来
轻松的完成这项任务。

	#include <netlink/addr.h>

	int nl_addr_iszero(struct nl_addr *addr);

<H1>9.1.1 地址前缀长度</H1>

虽然从功能性上来说它属于只和路由相关的功能，但是它是在核心库[^1]中实现的。地址
可以绑定一个前缀长度，这个长度表示只有前面的 `n` 个位是重要的。这可以用来实现
诸如子网之类的功能。

使用 `nl_addr_set_prefixlen()` 和 `nl_addr_get_prefixlen()` 函数来操控前缀长度。

	#include <netlink/addr.h>

	void nl_addr_set_prefixlen(struct nl_addr *addr, int n);
	unsigned int nl_addr_get_prefixlen(struct nl_addr *addr);

**注意：** 默认的前缀长度被设置为（地址长度 × 8）

**地址辅助函数**

库中还存在其他的辅助函数帮助处理地址问题，`nl_addr_cmp()` 函数会忽略前缀长度
比较两个地址，并返回一个小于、等于或者大于 `0` 的整数。如果你需要考虑前缀长度的话，
你可以使用 `nl_addr_cmp_prefix()`。

	#include <netlink/addr.h>

	int nl_addr_cmp(struct nl_addr *addr, struct nl_addr *addr);
	int nl_addr_cmp_prefix(struct nl_addr *addr, struct nl_addr *addr);

如果你需要把地址呈现给用户，那么你需要以一种易于人类阅读的格式呈现它，这个格式对于
不同的地址簇来说是不一样的。`nl_addr2str()` 函数会通过在内部调用合适的转换函数来
应对这一点。它接收一个 `size` 长的 `buf` 来存储输出的字符串并返回 `buf` 的指针以
方便 `printf()` 函数的调用。

	#include <netlink/addr.h>

	char *nl_addr2str(struct nl_addr *addr, char *buf, size_t size);

如果不知道地址簇的话，地址数据会以十六进制的格式打印出来 `AA:BB:CC:DD:..`

通常唯一能够得知地址簇的方式是查看地址的长度。`nl_addr_guess_family()` 函数使用
这种方式基于地址长度返回猜测的地址簇。

	#include <netlink/addr.h>

	int nl_addr_guess_family(struct nl_addr *addr);

在分配地址之前你可能会想要检查给出的字符序列是否真的能代表你期望的地址簇中的一个
合法地址。`nl_addr_valid()` 函数可以用来完成这一个任务，如果给出的是地址簇中的
合法地址，这个函数会返回 `1`。关于合法的地址格式的更多信息，请查看 `inet_pton(3)`
、`dnet_pton(3)`。

	#include <netlink/addr.h>

	int nl_addr_valid(char *addr, int family);

<H2 id = "abstract_data">9.2. 抽象数据</H2>

抽象数据类型是一种简单的数据类型，它的主要目的是简化任意长度的 `netlink` 属性的
使用。

**数据对象的分配**

`nl_data_alloc()` 函数会分配一个新的抽象数据对象，然后使用提供的数据填充它。
`nl_data_alloc_attr()` 完成同一项工作，只不过它是基于 `netlink` 属性的载荷中的
数据的。新的数据对象也可以通过调用 `nl_data_clone()` 函数克隆已有的数据而得到。

	struct nl_data *nl_data_alloc(void *buf, size_t size);
	struct nl_data *nl_data_alloc_attr(struct nlattr *attr);
	struct nl_data *nl_data_clone(struct nl_data *data);
	void nl_data_free(struct nl_data *data);

**访问数据**

`nl_data_get()` 函数返回一个指向数据的指针，数据的大小是通过 `nl_data_get_size()`
函数返回的。

	void *nl_data_get(struct nl_data *data);
	size_t nl_data_get_size(struct nl_data *data);

**数据处理的辅助函数**

`nl_data_append()` 函数会重新分配内部的数据缓冲区，然后把给定的 `buf` 中的数据
添加到已有的数据末尾。

	int nl_data_append(struct nl_data *data. void *buf, size_t size);

**注意：** 任何 `nl_data_append()` 函数的调用都会使得所有 `nl_data_get()` 函数
返回的同一数据对象的指针失效。

	int nl_data_cmp(struct nl_data *data, struct nl_data *data);


version 3.2

最后一次更新事件是 2014-07-16 12:04:02 CEST

***

  [^1]: `libnl` 库分为多个子库，请参考文档的[第一部分][partone]，这份文档是核心库的开发者文档。【译者注】

  [partone]: /netlink/2015/01/20/libnl-translation-part1.html
