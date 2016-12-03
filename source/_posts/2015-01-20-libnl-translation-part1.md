---
author: 郭荣飞
date: 2015-01-20
title: Netlink 库 -- 官方开发者教程中文版第一部分
tags:
 - netlink
 - libnl
---


<H1 id = "introduction">1. 引言</H1>


核心库（core library）提供了使用 netlink 套接字进行通信的基础功能。它处理套接
字的连接建立和断开、发送和接收数据、构造和解析消息、提供可配置的接收状态机。除此
之外它还提供了一套抽象数据类型的框架，这套框架使得基于对象的 netlink 协议实现
起来更加的简单，在这种协议中，对象可以通过基于 netlink 的协议来添加、删除、
或者修改。

<!--more-->

<H3>库的层次结构</H3>


这工程被分割成了下面几个库：

![结构层次图][lib_overview]

  [lib_overview]: http://www.infradead.org/~tgr/libnl/doc/images/library_overview.png

**Netlink Library(libnl)**

套接字的处理、发送和接收数据、消息的构造和解析、......

**Routing Family Library(libnl-route)**

地址、链路、邻居节点、路由、流量控制、邻居节点表、......

**Netfilter Library(libnl-nf)**

连接追踪、记录日志、排队（queueing）

**Generice Netlink Library(libnl-genl)**

控制器的 API，协议簇（family）和命令的注册


<H2 id = "how_to_read">1.1. 如何阅读这份文档</H2>


这些库提供了大量的 API，通常大部分的应用程序都只需要用到其中的一小部分。有些用户
可能只会关心低层（low level）的 netlink 消息处理 API，而其他用户可能主要使用高层
（high level）的 API，这些主要依赖于你的应用程序的类型。

无论你是属于那种情况，我们都推荐你首先熟悉下 netlink 协议。

  - [Netlink 协议基础][nlpf]

  [nlpf]: /netlink/2015/01/23/libnl-translation-part2.html#nlp_fundamentals

低层 API 在下面两部分中有详细介绍：

  - [Netlink 套接字][netlink_socket]

  - [发送和接收消息/数据][send_recv_msg_data]

  [netlink_socket]: /netlink/2015/01/26/libnl-translation-part3.html#netlink_sock
  [send_recv_msg_data]: /netlink/2015/01/27/libnl-translation-part4.html#snd_rcv_msg_data

<H2 id = "how_to_link">1.2. 如何链接到这个库</H2>

<H3>使用 autoconf 检查库是否存在</H3>

那些使用 autoconf 的项目可以使用 PKG_CHECK_MODULES() 来检测系统中是否存在某个特定
版本的 libnl。下面这个例子同时也展示了如何取得链接到 libnl 库所需要的 CFLAGS 和
链接依赖。

下面这个例子展示 了如何检查特定版本的 libnl 是否存在。如果存在，这个例子也展示了
如何正确的扩展 CFLAGS 和 LIBS 变量：


```cpp
PKG_CHECK_MODULES(LIBNL3,libnl3-3.0>=3.1,[have_libnl3]=yes,[have_libnl3=no])
if (test "$(have_libnl3)"="yes"); then
    CFLAGS+="$(LIBNL3_CFLAGS)"
    LIBS+="$(LIBNL3_LIBS)"
fi
```

**注意：** pkgconfig 被命名成 libnl-3.0.pc 是遗留问题，它实际上也包含了版本号大于
3.1 的库。

<H3>头文件</H3>


需要包含的头文件主要是 <netlink/netlink.h> 这个文件。根据你使用的子系统和组件的
不同，你可能还需要在你头文件中添加一些额外的头文件。

```cpp
#include <netlink/netlinl.h>
#include <netlink/cache.h>
#include <netlink/route/link.h>
```


<H3>依赖于版本号的代码</H3>


如果你希望能在你的代码中链接 libnl 的多个版本，你可以让编译器根据你想要链接的
libnl 库的特定版本来编译你代码中包含的特定部分的代码。


```cpp
#include <netlink/version.h>

#if LIBNL_VER_NUM >= LIBNL_VER(3.1)
    /* include code if compiled with libnl version >= 3.1 */
#end if
```


<H3>链接</H3>

```cpp
$gcc myprogram.c -o myprogram $(pkgconfig --cflags --libs libnl-3.0)
```


<H2 id = "debugging">1.3. 调试</H2>

这个库在编译的时候包含了调试语句，这使得它可以在你把 NLDBG 这个环境变量的值设置为
> 0 的值的时候往 stderr 中打印调试信息。


```cpp
$ NLDBG=2 ./myprogram
```



<table rules="all" style="margin-left:auto; margin-right:auto;"
	width="80%" frame="border" cellspacing="0" cellpadding="4">
	<caption class="title">表 1. 调试级别</caption>
	<col width="17%" />
	<col width="83%" />
	<thead>
		<tr>
			<th align="left" valign="top"> 级别 </th>
			<th align="left" valign="top"> 描述 </th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td align="left" valign="top"><p class="table">0</p></td>
			<td align="left" valign="top"><p class="table">
				关闭调试 (默认)
			</p></td>
		</tr>
		<tr>
			<td align="left" valign="top"><p class="table">1</p></td>
			<td align="left" valign="top"><p class="table">
				警告信息、重要的事件和通知信息
			</p></td>
		</tr>
		<tr>
			<td align="left" valign="top"><p class="table">2</p></td>
			<td align="left" valign="top"><p class="table">
				多一些更不重要的信息
			</p></td>
		</tr>
		<tr>
			<td align="left" valign="top"><p class="table">3</p></td>
			<td align="left" valign="top"><p class="table">
				导致调试信息刷屏重复性事件
			</p></td>
		</tr>
		<tr>
			<td align="left" valign="top"><p class="table">4</p></td>
			<td align="left" valign="top"><p class="table">
				比上面的信息还更不重要的消息
			</p></td>
		</tr>
	</tbody>
</table>


<H3>Netlink 协议的调试</H3>


通常查看套接字之间交换的 netlink 消息流是非常有用的。把环境变量 NLCB 的值设置为
debug（NLCB=debug）可以运行调试消息处理器，它会把交换的 netlink 消息打印成易于
我们阅读的格式并输出到 stderr 上。


	$ NLCB=debug ./myprogram
	-- Debug: Sent Message:
	--------------------------   BEGIN NETLINK MESSAGE ---------------------------
	  [HEADER] 16 octets
	    .nlmsg_len = 20
	    .nlmsg_type = 18 &lt;route/link::get&gt;
	    .nlmsg_flags = 773 &lt;REQUEST,ACK,ROOT,MATCH&gt;
	    .nlmsg_seq = 1301410712
	    .nlmsg_pid = 20014
	  [PAYLOAD] 16 octets
	    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00       ................
	---------------------------  END NETLINK MESSAGE   ---------------------------
	-- Debug: Received Message:
	--------------------------   BEGIN NETLINK MESSAGE ---------------------------
	  [HEADER] 16 octets
	    .nlmsg_len = 996
	    .nlmsg_type = 16 &lt;route/link::new&gt;
	    .nlmsg_flags = 2 &lt;MULTI&gt;
	    .nlmsg_seq = 1301410712
	    .nlmsg_pid = 20014
	  [PAYLOAD] 16 octets
	    00 00 04 03 01 00 00 00 49 00 01 00 00 00 00 00       ........I.......
	  [ATTR 03] 3 octets
	    6c 6f 00                                              lo.
	  [PADDING] 1 octets
	    00                                                    .
	  [ATTR 13] 4 octets
	    00 00 00 00                                           ....
	  [ATTR 16] 1 octets
	    00                                                    .
	  [PADDING] 3 octets
	    00 00 00                                              ...
	  [ATTR 17] 1 octets
	    00                                                    .
	  [...]
	---------------------------  END NETLINK MESSAGE   ---------------------------




