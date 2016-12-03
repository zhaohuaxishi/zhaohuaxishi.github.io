---
author: 郭荣飞
date: 2015-02-15
title: Netlink 库 -- 官方开发者教程中文版第七部分
tags:
 - netlink
 - libnl
---


<H1 id = "callback_config">7. 回调配置</H1>

为了控制一些函数的行为， `libnl` 库在许多地方都设置了回调函数挂钩点并允许某些函数
的覆写。所有的回调和覆写函数都是封装在 `struct nl_cb` 结构体中，这个结构存在于
`netlink` 套接字中或者作为函数的参数直接传递。

<!--more-->

<H2 id = "callback_hooks">7.1. 回调函数挂钩点</H2>

回调函数挂钩点散布在库中的各个角落中以提供消息处理的入口，同时它们也用于响应某些
事件。

回调函数可能会返回下面这些返回值：

<table rules="all" width="100%" frame="border" cellspacing="0" cellpadding="4">
	<colgroup>
		<col width="20%">
		<col width="80%">
	</colgroup>
	<thead>
		<tr>
			<th align="left" valign="top"> 返回值      </th>
			<th align="left" valign="top"> 描述</th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td align="left" valign="top">
				<p class="table"><code>NL_OK</code></p>
			</td>
			<td align="left" valign="top">
				<p class="table">已经处理.</p>
			</td>
		</tr>
		<tr>
			<td align="left" valign="top">
				<p class="table"><code>NL_SKIP</a></code></p>
			</td>
			<td align="left" valign="top">
				<p class="table">跳过当前处理的消息继续解析接收缓冲区中下一个消息的处理。</p>
			</td>
		</tr>
		<tr>
			<td align="left" valign="top">
				<p class="table"><code>NL_STOP</a></code></p>
			</td>
			<td align="left" valign="top">
				<p class="table">停止解析并丢弃接收缓冲区中的所有数据。</p>
			</td>
		</tr>
	</tbody>
</table>

**回调函数的默认实现**

这个库提供了回调函数集的三个默认实现：默认的 *NL_CB_DEFAULT 集合。它是默认的处理流程。
*NL_CB_VERBOSE 集合，这个函数集基于默认的函数集但是它会把错误消息，非法消息，
泛滥（`overrun`）的消息和没有处理的合法消息的错误信息输出到 `stderr` 上。`nl_cb_set()` 函数
和 `nl_cb_err()` 函数参数中的指针可以用来提供一个 `FILE*` 用来覆盖 `stderr`。
*NL_CB_DEBUG 函数集则是出于调试目的而实现的。这个函数集基于 `verbose` 集合但是
它会解码和复制发送或接收到的每一个消息，并把数据发送了控制台（`console`）。

**nl_sendmsgs() 回调函数挂钩点：**

<table rules="all" width="100%" frame="border" cellspacing="0" cellpadding="4">
	<colgroup>
		<col width="28%">
		<col width="57%">
		<col width="14%">
	</colgroup>
	<thead>
		<tr>
			<th align="left" valign="top"> 回调 ID        </th>
			<th align="left" valign="top"> 描述                       </th>
			<th align="left" valign="top"> 默认返回值</th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td align="left" valign="top"><p class="table"><code>NL_CB_MSG_OUT</code></p></td>
			<td align="left" valign="top"><p class="table"><em>发送出去的每一个消息</em></p></td>
			<td align="left" valign="top"><p class="table"><code>NL_OK</code></p></td>
		</tr>
	</tbody>
</table>

任何被 `NL_CB_MSG_OUT` 调用的函数都会可以返回一个负的错误码以达到阻止消息发送
出去并返回错误码的目的。

**nl_recvmsgs() 回调函数挂钩点（按照优先级排序）：**

<table rules="all" width="100%" frame="border" cellspacing="0" cellpadding="4">
	<colgroup>
		<col width="28%">
		<col width="57%">
		<col width="14%">
	</colgroup>
	<thead>
		<tr>
			<th align="left" valign="top"> 回调 ID</th>
			<th align="left" valign="top"> 描述</th>
			<th align="left" valign="top"> 默认返回值</th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td align="left" valign="top"><p class="table"><code>NL_CB_MSG_IN</a></code></p></td>
			<td align="left" valign="top"><p class="table"><em>接收到的每一个消息</em></p></td>
			<td align="left" valign="top"><p class="table"><code>NL_OK</a></code></p></td>
		</tr>
		<tr>
			<td align="left" valign="top"><p class="table"><code>NL_CB_SEQ_CHECK</a></code></p></td>
			<td align="left" valign="top"><p class="table"><em>可以重新设置序列号检查算法</em></p></td>
			<td align="left" valign="top"><p class="table"><code>NL_OK</a></code></p></td>
		</tr>
		<tr>
			<td align="left" valign="top"><p class="table"><code>NL_CB_INVALID</a></code></p></td>
			<td align="left" valign="top"><p class="table"><em>非法消息</em></p></td>
			<td align="left" valign="top"><p class="table"><code>NL_STOP</a></code></p></td>
		</tr>
		<tr>
			<td align="left" valign="top"><p class="table"><code>NL_CB_SEND_ACK</a></code></p></td>
			<td align="left" valign="top"><p class="table"><em>含有 NLM_F_ACK 标志的消息</em></p></td>
			<td align="left" valign="top"><p class="table"><code>NL_OK</a></code></p></td>
		</tr>
		<tr>
			<td align="left" valign="top"><p class="table"><code>NL_CB_FINISH</a></code></p></td>
			<td align="left" valign="top"><p class="table"><em>类型为 NLMSG_STOP 的消息</em></p></td>
			<td align="left" valign="top"><p class="table"><code>NL_STOP</a></code></p></td>
		</tr>
		<tr>
			<td align="left" valign="top"><p class="table"><code>NL_CB_SKIPPED</a></code></p></td>
			<td align="left" valign="top"><p class="table"><em>类型为 NLMSG_NOOP 的消息</em></p></td>
			<td align="left" valign="top"><p class="table"><code>NL_SKIP</a></code></p></td>
		</tr>
		<tr>
			<td align="left" valign="top"><p class="table"><code>NL_CB_OVERRUN</a></code></p></td>
			<td align="left" valign="top"><p class="table"><em>em>类型为 NLMSG_OVERRUN 的消息</em></p></td>
			<td align="left" valign="top"><p class="table"><code>NL_STOP</a></code></p></td>
		</tr>
		<tr>
			<td align="left" valign="top"><p class="table"><code>NL_CB_ACK</a></code></p></td>
			<td align="left" valign="top"><p class="table"><em>ACK 消息</em></p></td>
			<td align="left" valign="top"><p class="table"><code>NL_STOP</a></code></p></td>
		</tr>
		<tr>
			<td align="left" valign="top"><p class="table"><code>NL_CB_VALID</a></code></p></td>
			<td align="left" valign="top"><p class="table"><em>所有的合法消息</em></p></td>
			<td align="left" valign="top"><p class="table"><code>NL_OK</a></code></p></td>
		</tr>
	</tbody>
</table>

所有的这些函数都可能返回 `NL_OK`,`NL_SKIP`,`NL_STOP`。

消息处理回调函数是通过 `nl_cb_set()` 函数设置的：

	#include <netlink/handlers.h>

	int nl_cb_set(struct nl_cb *cb, enum nl_cb_type type, emum nl_cb_kind kind,
		nl_recvmsg_msg_cb_t func, void *arg);

	typedef int(*nl_recvmsg_msg_cb_t)(struct nl_msg *msg, void *arg);

**错误消息的回调函数**

错误消息的回调函数挂钩点使用了一个特殊的函数原型：

	#include <netlink/handler.h>

	int nl_cb_err(struct nl_cb *cb, enum nl_cb_kind, nl_recvmsg_err_cb_t func, void *arg);

	typedef int(* nl_recvmsg_err_cb_t)(struct sockaddr_nl *nla, struct nlmsgerr *nlerr, void *arg);

**例子：设置回调函数集合**

	#include <netlink/handlers.h>

	/* 分配一个回调函数集合并且通过默认的 verbose 接合来初始化它 */
	struct nl_cb *cb = nl_cb_alloc(nl_cb_verbose);

	/* 修改函数集合让它多所有合法的消息都调用 my_func() */
	nl_cb_set(cb, NL_CB_VALID, NL_CB_CUSTOM, my_func, NULL);

	/*
	 * 设置错误消息处理器为 verbose 的默认实现，并设置它把所有的错误
	 * 输出到给定的文件描述符中。
	 */

	FILE *file = fopen(...);
	nl_cb_err(cb, NL_CB_VERBOSE, NULL, file);

<H2 id = "overwrite_internal_fun">7.2. 内部函数的覆盖</H2>

当库需要通过高层（`high level`）接口来发送和接收 `neltink` 消息的时候，它实际上是
调用自己的低层（`low level`）接口来完成这一任务的。如果默认的特性并不符合应用程序
的需求的话，你可以使用自己的私有实现来覆盖许多内部函数。

**覆盖 `recvmsgs()` 函数**

关于库的内部什么时候以及如何调用 `recvmsgs()` 函数更多的信息，请参考
[接收 `netlink` 消息][recv_netlink_msg] 一节。

	#include <netlink/handlers.h>

	void nl_cb_overwrite_recvmsgs(struct nl_cb *cb,
		int (*func)(struct nl_sock *sk, struct nl_cb *cb));

如果想要你的 `recvmsgs()` 实现和高层接口相融合的话,下面这些条件必须得到满足：

  * **一定要**考虑回调配置 `cb`,因此：

  * **一定要**对所有的合法消息调用　`NL_CB_VALID` 回调函数

  * **一定要**对所有的 `ACK` 消息调用 `NL_CB_ACK` 回调函数

  * **一定要**正确的处理多段式消息，对于每一个消息直到收到了 `NLMSG_DONE` 才调用
    `NL_CB_VALID` 回调函数

  * **一定要**在收到了 `NLMSG_OVERRUN` 和 `NLMSG_ERROR` 消息的时候报告错误码。

**覆盖 `nl_recv()` 函数**

通常我们只需要覆盖负责从套接字中接收实际数据的 `nl_recv()` 函数而不需要完全替换整个
`recvmsgs()` 函数逻辑。

请查看 [接收特性][recv_characteristics] 以便获得内部函数如何以及何时调用 `nl_recv()`
函数。

	#include <netlink/handlers.h>

	void nl_cb_overwrite_recv(struct nl_cb *cb,
		int (*func)(struct nl_sock *sk,
			struct sockaddr_nl *addr,
			unsigned char **buf,
			struct ucred **cred))

一个私有的 `nl_recv()` 函数实现必须满足下面这些条件：

   * **一定要**返回读取的字节数目或者是在遇到错误时返回负的错误码。这个函数也可以返回
     `0` 表示没有读取到任何的数据。

   * **一定要**把 `*buf` 设置成一个包含读取到的数据的缓冲区。实现必须保证调用者可以
     从中安全的读取作为返回值的字节数。

   * **可以**把 `addr` 设置成发送数据的对等方的 `netlink` 地址。

   * **可以**把 `*cred` 设置成一个新分配的包含信用值的 `struct ucred` 结构体。

**覆盖 `nl_send()` 函数**

关于库的内部如何以及何时调用 `nl_send()` 函数，请参考 [发送 `netlink` 消息][send_netlink_msg]
一节。

	#include <netlink/handlers.h>

	void nl_cb_overwrite_send(struct nl_cb *cb, int(*func)(struct nl_sock *sk,
					struct nl_msg *msg));

私有实现必须负责发送 `netlink` 消息，并在成功发送的时返回 `0` 或在错误时返回负的错误码。

   [recv_netlink_msg]: /netlink/2015/01/27/libnl-translation-part4.html#rcv_msg
   [recv_characteristics]: /netlink/2015/01/27/libnl-translation-part4.html#parse_character
   [send_netlink_msg]: /netlink/2015/01/27/libnl-translation-part4.html#send_msg
