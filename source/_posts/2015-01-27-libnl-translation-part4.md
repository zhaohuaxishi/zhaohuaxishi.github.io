---
author: 郭荣飞
date: 2015-01-27
title: Netlink 库 -- 官方开发者教程中文版第四部分
tags:
 - netlink
 - libnl
---

<H1 id = "snd_rcv_msg_data">4. 消息或数据的发送和接收</H1>

<H2 id = "send_msg">4.1 发送消息</H2>

使用 netlink 套接字发送一条 netlink 消息的标准方式是调用 `nl_send_auto()` 函数。
这个函数会通过填充缺失的字段以及填充 netlink 消息头部自动完成这条 netlink 消息的
构建。此外这个函数还会根据 netlink 套接字设置的选项和地址来处理寻址问题。之后这
个消息会被传递给 nl_send() 函数。

<!--more-->

如果 `nl_send()` 实现的默认函数逻辑并不适合你的应用程序，你可以通过调用
`nl_cb_overwirte_send()` 来指定你自己的实现方式以覆盖原本的接收函数 `nl_send()`

	nl_send_auto(sk, msg)
		+
		|
		|
		+-------->nl_complete(sk, msg)
		|
		|
		|        通过 nl_cb_overwrite_send() 指定的接收函数
		+-----------------------------------+
		|                                   |
		v                                   v
	   nl_send(sk, msg)                     send_func()

<H3>使用 nl_send()</H3>

如何你不需要任何的消息自动补全功能，你可以直接使用 `nl_send()` 函数，但是需要注
意的是，`libnl` 库内部调用 `nl_send_auto` 来发送数据的时候还是会调用 `nl_send()`
函数。因此如果你想要使用任何的高层（higher level）的接口但又不喜欢 `nl_send()`
的的行为的话，你需要调用 `nl_cb_overwrite_send()` 函数来覆写 `nl_send()` 函数。

`nl_send()` 函数的目的是把 `netlink` 消息嵌入到一个 `iovec` 结构体中然后把这个结
构体传递给 `nl_send_iovec()` 函数。

	nl_send(sk, msg)
	    |
	    v
	nl_send_iovec(sk, msg, iov, iovlen)

<H3> 使用 nl_send_iovec()函数</H3>

`nl_send_iovec()` 接收一个最终的 `netlink` 消息然后用它来填充一个用来发送的
`struct msghdr` 结构。这个函数首先查看 `struct nl_msg` 结构是否设置了特殊的对等
端（查看 `nlmsg_set_dst()` 函数）。如果没有的话，它会尝试取得在套接字中设定的对
等端地址（查看 `nl_socket_peer_port()`）。如果上面两个都没有找到地址的的话，这个
消息会以非定址（unaddressed）状态发送，把寻找正确的对等端的任务交给内核。

如果存在并且启用了信用度机（credentials）制的话 `nl_send_iovec()` 还会增加信用度
，更多信息请查看 [core_sk_cred][sk_cred]。

接下来消息会被传递给 nl_sendmsg()。

	nl_send_iovec(sk, msg, iov, iovlen);
	    |
	    v
	nl_sendmsg(sk, msg, msghdr)

<H3>使用 nl_sendmsg()</H3>

`nl_sendmsg()` 函数接收一个最终的 `netlink` 消息和一个可能包含对等节点地址的
`struct msghdr` 结构体[^1]。这个函数会把套接字中定义的本地端口（参考
`nl_socket_set_local_port()`）拷贝到 `netlink` 消息的头部中去。

至此消息构造完成，可以准备发送出去了。

	nl_sendmsg(sk, msg, msghdr)
	    |
	    |---------------------------+
	    |				v
	    |			NL_CB_MSG_OUT()
	    |<--------------------------+
	    v
	sendmsg()

在发送之前，应用程序还有最后一次机会更改消息。这个消息会被传递给 `nl_cb_msg_out`
回调函数，这个函数可能会检查或者修改消息并返回一个错误码。如果错误码是 `nl_ok`
这个消息会通过 `sendmsg()` 函数发送出去，这个函数的返回值是发送的字节数。否则的
话中止这个消息的发送处理并且返回回调函数提供的错误码。关于如何设置回调函数请参考
[修改套接字回调配置][cbk_cfg_modify] 一节。

<H3>使用 nl_sendto()发送原始数据</H3>

如果你希望通过 netlink 套接字发送一个原始数据，下面这个函数会把提供给它的任何缓
冲区直接传递给 `sendto()` 函数：

	#include <netlink/netlink.h>

	int nl_sendto(struct nl_sock *sk, void *buf, size_t size);

<H3> 发送简单消息</H3>

`libnl` 库中存在一个特殊的接口用来发送简单的消息。这个函数接收 `netlink` 消息类
型、可选的 `netlink` 消息标志以及可选的数据缓冲区和数据长度。

	#include <netlink/netlink.h>

	int nl_send_simple(struct nl_sock *sk, int type, int flags,
			void *buf, size_t size);

这个函数会根据提供的消息的类型和标志构造一个 `netlink` 消息头部然后把数据缓冲区
作为消息的有效负载。新构建的消息会调用 `nl_send_auto()` 发送出去。

在下面这个例子[^2]中，我们将会发送一个请求消息，这个消息会导致内核复制一份所有的
网络链路的列表到用户空间。

	#include <netlink/netlink.h>

	struct nl_sock *sk;
	struct rtgenmsg rt_hdr = {
		.rtgen_family = AF_UNSPEC,
	};

	sk = nl_socket_alloc();
	nl_connect(sk, NETLINK_ROUTE);

	nl_send_simple(sk, RTM_GETLINK, NLM_F_DUMP, &rt_hdr, sizeof(rt_hdr));

<H2 id = "rcv_msg">接收消息</H2>

接收 `netlink` 消息最简单的方式就是调用 `nl_recvmsgs_default()` 函数。它会根据套
接字中定义的场景来接收消息。虽然这个函数的默认行为对大多数的应用程序都适用，你的
应用程序也可以调整其中的细节。

`nl_recvmsgs_default()` 这个函数也会在 `libnl` 库内部需要接收和解析 `netlink` 消
息的时候被调用。

这个函数会读取在套接字中的回调配置，然后调用 `nl_recvmsgs()` 函数：

	nl_recvmsgs_default(sk)
	    |
	    |  cb = nl_socket_get_cb(sk)
	    v
	nl_recvmsgs(sk, cb)

<H3>使用 nl_recvmsgs() 函数</H3>

`nl_recvmsgs()` 函数实现了实际的接收循环，除非这个套接字被设置成了非阻塞模式，否
则的话这个函数会一直阻塞直到接收到 `netlink` 消息为止。

在一些极端的场景下，某些接收特性没办法通过修改回调配置（参考 [修改套接字回调配置
][cbk_cfg_modify]一节）调整内部的消息接收函数这种方式来获得。这时应用程序可以通
过调用 `nl_cb_overwrite_recvmsgs()` 函数提供自己的完整的实现方案以便覆盖所有的
`nl_recvmsgs()` 函数的调用。

	nl_recvmsgs(sk, cb)
	    |
	    |   通过 nl_cb_overwrite_recvmsgs() 提供的自己的消息接收函数
	    |-----------------------------------+
	    v					v
	internal_recvmsgs()		my_recvmsgs()

<H3 id = "recv_char">接收特性</H3>

如果应用程序没有通过 `nl_cb_overwrite_recvmsgs()` 提供自己的 `recvmsgs()` 函数的
实现。从一个 `netlink` 套接字接收数据将会有下面这些特性。


		internal_recvmsgs()
			|
	+-------------->|     使用 nl_cb_overwrite_recv() 函数指定的接收函数
	|               |- - - - - - - - - - - - - - - -
	|               v                              v
	|           nl_recv()                      my_recv()
	|               |<- - - - - - - - - - - - - - -+
	|               |<- - - - - - -+
	|               v              | 是否还有更多的消息需要解析? (nlmsg_next())
	|         Parse Message        |
	|               |- - - - - - - +
	|               v
	+------- NLM_F_MULTI set?
			|
			v
		    (SUCCESS)


首先调用函数 `nl_recv()` 函数从 `netlink` 套接字中接收数据。这个函数也可能被应用
程序调用 `nl_cb_overwrite_recv()` 提供的私有实现覆盖掉。如果 `netlink` 字节流实
际上是从文件或者其他数据源读取到而不是直接从套接字接收下来的话，这种覆盖是非常有
用的。

如果数据已经被读取出来了的话，`internal_recvmsgs()` 函数将会尝试解析数据。这个解
析过程会一直重复直到遇到 `NL_STOP` 类型的数据或者遇到错误又或者是没有可以解析的
数据为止。

如果最后一个成功解析的消息是一个多段式数据（参考 [分段式数据][multi_part_msg]）
并且解析器没有因为遇到错误或者接收到 `NL_STOP` 消息而退出的话，`nl_recv()` 函数
或者应用程序的实现的私有接收函数会被再次调用，而解析器也会再次启动。

关于如何从解析器中取出有效的 `netlink` 消息以及如何控制解析器的行为的详细信息请
参考 [[core_parse_character]][parse_character]。

<H3 id = "parse_character">消息解析特性</H3>

从一个 `netlink` 套接字中读取下来每一个 `netlink` 消息都会调用内部的解析器进行解
析。它通常是被 `nl_recv()` 调用（参考 [[core_recv_character]][recv_char]）。

解析器首先确定数据流的长度足够容纳一个 `netlink` 消息头部，然后确定消息头部中提
供的消息长度没有超过数据流的总长。

如果上面的条件都满足的话，分配一个新的 `strutct nl_msg` 然后把消息传递给回调函数
`NL_CB_MSG_IN`（如果提供了的话）。和其他的回调函数一样，它可能会返回 `NL_SKIP`
来忽略当前的消息然后继续解析下一个消息也可能返回 `NL_STOP` 从而完全停止解析过程
。

下一步是检查消息的序列号和当前期望的序列号是否一致。应用程序可以通过把回调函数
`NL_CB_SEQ_CHECK` 设置成自己的私有实现来提供自己的序列号检验算法。事实上调用
`nl_socket_disable_seq_check()` 函数来取消序列号检测只不过是把 `NL_CB_SEQ_CHECK`
回调函数设置成了一个永远返回 `NL_OK` 的函数而已。

此外还存在一个回调函数挂钩点 `NL_CB_SEND_ACK`，这个函数会在收到含有 `NLM_F_ACK`
标志的消息之后被调用。虽然我并不知道有哪一个用户空间的 `netlink` 套接字需要给发
送方发送ACK消息，但是应用程序还是有可能会需要这么做（参考 [ACKs][acks] 一节）

		 parse()
		  |
		  v
	      nlmsg_ok() --> Ignore
		  |
		  |- - - - - - - - - - - - - - - v
		  |                         NL_CB_MSG_IN()
		  |<- - - - - - - - - - - - - - -+
		  |
		  |- - - - - - - - - - - - - - - v
	     Sequence Check                NL_CB_SEQ_CHECK()
		  |<- - - - - - - - - - - - - - -+
		  |
		  |              Message has NLM_F_ACK set
		  |- - - - - - - - - - - - - - - v
		  |                      NL_CB_SEND_ACK()
		  |<- - - - - - - - - - - - - - -+
		  |
		Handle Message Type


<H2 id = "auto_ack"> Auto-ACK 模式</H2>

TODO


  [^1]: 这句话的原文是 `optioanl struct msgdhr containing the peer address` 从语法上翻译的话应该是说 `struct msghdr` 结构体是可选的。但是这本身是解释不通的，因为 `nl_sendmsg()` 这个函数的函数原型中参数 struct msghdr * hdr 是固定的参数而并非可选的，这一点可以参考API文档中的 [相关介绍][api_nl_sendmsg]。从上一段关于 `nl_send_iovec()` 函数的寻址处理中提到地址可能最终不确定，所以这里翻译成了msgdhr 可能包含地址，如果有更合理的解释，请联系我 <guorongfei@126.com>。「译者注」

  [^2]: 这个例子在的最后一条语句中 `nl_send_simple` 函数的第一个参数写成了 `sock`，个人认为这是笔误。

  [sk_cred]: /netlink/2015/01/26/libnl-translation-part3.html#sk_cred
  [multi_part_msg]: /netlink/2015/01/23/libnl-translation-part2.html#multipart_msg
  [parse_character]: #parse_character
  [recv_char]: #recv_char
  [acks]: /netlink/2015/01/23/libnl-translation-part2.html#acks
  [api_nl_sendmsg]: http://www.infradead.org/~tgr/libnl/doc/api/group__send__recv.html#gad9fed072e3e69c2860d4c8e7c991677b
  [cbk_cfg_modify]: /netlink/2015/01/26/libnl-translation-part3.html#callback_config
