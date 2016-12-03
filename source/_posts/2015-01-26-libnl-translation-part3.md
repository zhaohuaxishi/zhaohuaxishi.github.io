---
author: 郭荣飞
date: 2015-01-26
title: Netlink 库 -- 官方开发者教程中文版第三部分
tags:
 - netlink
 - libnl
---

<H1 id = "netlink_sock">3. Netlink 套接字</H1>

为了使用 netlink 协议，我们首先需要一个 netlink 套接字[^1]。每个套接字都定义了一
个独立的消息发送和接收的上下文环境。一个应用程序可以使用多个套接字，比如一个套接
字用于发送请求消息和接受应答消息另一个套接字用来订阅某个多播组以便接收通知消息。

<!--more-->

<H2 id = "sock_struct">3.1. 套接字结构体（struct nl_sock）</H2>

netlink 套接字以及和他相关的包括实际的文件描述符在内的所有属性都使用 nl_sock
结构体表示的。

```cpp
#include <netlink/socket.h>

struct nl_sock *nl_socket_alloc(void)
void nl_socket_free(struct nl_sock)
```

应用程序必须为它想要使用的每一个 netlink 套接字分配一个 `struct nl_sock` 结构体
实例。

<H2 id = "sequence_num">3.2. 序列号</H2>

libnl 库会自动为应用程序管理序列号。在套接字结构体内部存储了一个序列号计数器，如
果你发送了一个期望得到回复的请求消息，而且这个回复是诸如错误消息或者是其他任何需
要用到相关的原始消息的消息，那么这个计数器会自动递增。

此外，这个计数器也可以直接通过 `nl_socket_use_seq()` 函数使用。这个函数会返回计
数器当前的值然后自增一。

```cpp
#include <netlink/socket.h>

void nl_socket_use_seq(struct nl_sock *sk);
```

当然大部分的应用程序都不会想要自己处理序列号，是调用 `nl_send_auto()` 函数的时候
，序列号会自动填充并在收到回复信息的时候再次匹配。更多信息请参考 [消息/数据的发
送和接收][msg_data_snd_rcv] 一节

如果 netlink 协议的实现不需要使用请求/应答模式，比如当一个套接字是用来接收通知消
息的时候，序列号的这种行为可以也必须取消掉.

```cpp
#include <netlink/socket.h>

void nl_socket_disable_seq_check(struct nl_sock *sk);
```

关于 nelink 序列号背后的的更多理论知识，请参考 [序列号][seq_num] 一节。

<H2 id = "multi_grp_sub">3.3. 多播组的订阅</H2>

每一个套接字都可以订阅它连接的 netlink 协议的任意数量的多播组，之后这个套接字将
会接收到发送到其中任何一组的任何一条消息的拷贝。多播组一般是用来实现事件通知。

在 2.6.14 版本之前的内核中组的订阅是通过一个位掩码（bitmask）来实现的，这就限制
了每个协议簇最多能够使用 32 个组。虽然在新的代码中不建议再使用这个过时的接口，你
还是可以通过 `nl_join_groups()` 来访问它。

```cpp
#include <netlink/socket.h>

void nl_join_groups(struct nl_sock *sk, int bitmask);
```

2.6.14 内核引入了新的实现方式，这使得你可以订阅几乎是无限数量的多播组。

```cpp
#include <netlink/socket.h>

int nl_socket_add_memberships(struct nl_sock *sk, int group, ...);
int nl_socket_drop_memberships(struct nl_sock *sk, int group, ...);
```

<H3 id = "multi_grp_example">3.3.1 多播示例</H3>

```cpp
#include <netlink/netlink.h>
#include <netlink/socket.h>
#include <netlink/msg.h>

/*
 * 每次 nl_recvmsgs_default() 每次收到有效消息都会调用这个函数
 */

static int my_func(struct nl_msg *msg, void *arg)
{
    return 0;
}

struct nl_sock *sk;

/* 分配新的套接字 */

sk = nl_socket_alloc();

/*
 * 通知消息不是用序列号，取消序列号的自动检测
 */

nl_socket_disable_seq_check(sk);

/*
 * 定义一个在每次收到通知消息都会调用的回调函数
 */

nl_socket_modify_cb(sk, NL_CB_VALID, NL_CB_CUSTOM, my_func, NULL);

/* 连接到路由 netlink 协议 */

nl_connect(sk, NETLINK_ROUTE);

/* 订阅链路通知组 */

nl_socket_add_memberships(sk, RTNLGRP_LINK, 0);

/*
 * 开始接收消息。函数 nl_recvmsgs_default() 在接收到一个或者多个消息（通知）
 * 之前会一直阻塞，接收到的消息会被传递给 my_func() 回调函数。
 */

while (1)
    nl_recvmsgs_default(sock);

```cpp

<H2 id = "callback_config">3.4. 修改套接字回调配置</H2>

关于回调拦截点和它的重写能力请参考 [回调函数配置][cbk_cfg] 一节。

每一个套接字都有一个控制套接字行为的回调配置。这在某些情况下是必须要这么做的，
比如需要给每个套接字单独定义消息接收函数时候。当然在套接字之间共享回调配置也是
完全合法的操作。

下面这些函数可以用来读取或者设置一个套接字的回调配置：

```cpp
#include <netlink/socket.h>

struct nl_cb *nl_socket_get_cb(const struct nl_sock *sk);
void nl_socket_set_cb(struct nl_sock *sk, struct nl_cb *cb);
```


此外还有一个直接修改套接字回调配置的快捷函数：

```cpp
#include <netlink/socket.h>

int nl_socket_modify_cb(struct *sk, enum nl_cb_type type, enum nl_cb_kind kind,
        nl_recvmsgs_msg_cb_t func, void *arg);
```

**示例**

```cpp
#include <netlink/socket.h>

// 对套接字 sk 接收到的所有有效的消息都调用 my_input() 函数
nl_socket_set_cb(sk, NL_CB_VALID, NL_CB_CUSTOM, my_input, NULL);
```


<H2 id = "sock_attri">3.5. 套接字属性</H2>

<H3>本地端口号</H3>

本地端口唯一的标识了一个套接字的同时也被用来在寻址的时候锁定该套接字。一个本地端
口号是在分配套接字的时候自动生成的。它由进程ID（22位）和一个随机数（10位）组合而
成，所以每个进程可以使用1024个套接字。

```cpp
#include <netlink/socket.h>

uint32_t nl_socket_get_local_port(const struct nl_sock *sk);
void nl_socket_set_local_port(struct nl_sock *sk, uint32_t port);
```

关于端口号的更多的信息，请查看 [寻址][addressing] 一节。

**注意：** 你可以重新设置端口，但是你必须要保证你提供的值是唯一的，此外你还需要
保证没有其他的应用程序的套接字使用了这个值。

<H3>对等端口</H3>

一个套接字可以设置一个对等端口，这将会导致所有通过这个套接字发送的单播消息都会发
送到这个对等节点。如果没有设置对等节点，消息会被发送到内核，而内核会自动尝试把这
个套接字和内核里面同一个 netlink 协议簇下的套接字绑定在一起。人们通常不会把套接
字绑定到一个对等端口，因为通常一个在同一个 netlink 协议簇中只存在一个内核端套接
字。

```cpp
#include <netlink/socket.h>

uint32_t nl_socket_get_peer_port(const struct nl_sock *sk)
void nl_socket_set_peer_port(struct nl_sock *sk, uint32_t port);
```

关于端口号的更多的信息，请查看 [寻址][addressing] 一节。


<H3>文件描述符</H3>


Netlink 使用的是 BSD 套接字的接口，所以在每个套接字背后都有一个你可以直接使用的
文件描述符。

```cpp
#include <netlink/socket.h>

int nl_socket_get_fd(const struct nl_sock *sk);
```

如果一个套接字只是用来接收通知消息的，那么最好把它设置成非阻塞模式，然后周期性的
轮询新的通知消息。

```cpp
#include <netlink/socket.h>

int nl_socket_set_nonblocking(const struct nl_sock *sk);
```

<H3>发送和接收缓存区大小</H3>

套接字的缓冲区被用来在发送方和接收方之间充当 netlink 消息队列。这个缓冲区的大小
决定了你可以调用 write() 函数向 netlink 套接字写入的最大长度，也就是说它间接的定
义了最大消息的长度。这个缓冲区的默认大小是 32KiB。

```cpp
#include <netlink/socket.h>

int nl_socket_set_buffer_size(struct nl_sock *sk, int rx, int tx);
```

<H3 id = "sk_cred">Enable/Disable Credentials</H3>

TODO [^2]

```cpp
#include <netlink/socket.h>

int nl_socket_set_passcred(struct nl_sock *sk, int state);
```

<H3>启用/禁用 Auto-ACK 模式</H3>

下面的这些函数可以用来在一个套接字中启用或是禁用 `Auto-Ack` 模式。关于这个模式的
含义的详细信息请参考 [Auto-Ack 模式][auto_ack_mode] 一节。`Auto-Ack` 模式是默认
开启的。

```cpp
#include <netlink/socket.h>

void nl_socket_enable_auto_ack(struct nl_sock *sk);
void nl_socket_disable_auto_ack(struct nl_sock *sk);
```

<H3>启用/禁用消息窥视</H3>

如果启用了消息窥视功能，`nl_rcv()` 函数将会尝试使用 `MSG_PEEK` 来读取收到的下一
个消息的大小，并分配一个同样大小的缓存区。消息窥视功能是默认启用的但你可以使用下
面的函数禁用它：

```cpp
#include <netlink/socket.h>

void nl_socket_enable_msg_peek(struct nl_sock *sk);
void nl_socket_disable_msg_peek(struct nl_sock *sk);
```

<H3>启用/禁用数据包信息的接收</H3>

如果这项功能启用了的话，从内核中收到的每一条 netlink 消息都会在控制消息中额外包
含一个 `struct nl_pkinfo` 结构体。下面这些函数可以用来启用或禁用数据包信息的接收
。

```cpp
#include <netlink/socket.h>

int nl_socket_recv_pktinfo(struct nl_sock *sk, int state);
```

**注意：** `NETLINK_PKINFO` 的处理目前还没有实现。



  [^1]: 这篇文章中的套接字基本上都是指 `libnl` 库中的套接字结构体 `struct nl_sock` 而不是BSD 套接字接口中的套接字概念。
  [^2]: 在原文中这一段没有任何内容只有标题和下面的函数，所以很难判断这个函数的作用，个人也没有相关的知识背景所以没有翻译标题。「译者注」

  [msg_data_snd_rcv]: /netlink/2015/01/27/libnl-translation-part4.html#snd_rcv_msg_data
  [seq_num]: /netlink/2015/01/23/libnl-translation-part2.html#seq_num
  [cbk_cfg]: /netlink/2015/02/15/libnl-translation-part7.html#callback_config
  [addressing]: /netlink/2015/01/23/libnl-translation-part2.html#addressing
  [auto_ack_mode]: /netlink/2015/01/27/libnl-translation-part4.html#auto_ack


