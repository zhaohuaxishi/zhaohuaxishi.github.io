---
author: 郭荣飞
date: 2014-10-30
title: socket API 实现（五）—— connect 函数
tags:
 - Linux Kernel
 - socket
---

概述
====

通常在 CS 通信模式中，服务端通过 `socket->bind->listen->accept` 流程开始监听客户
端的连接。而在客户端则是通过 `socket->connect` 流程来建立和服务端的连接。我们在
前面的文章中已经分析过服务器端使用到的 4 个 API，这篇文章开始我们将会分析客户端
用到的 API。

首先要分析的第一个 API 是 `connect`，因为客户端创建 socket 的函数和服务端是一个
样的，都是使用 socket 函数。`connect` 函数的原型如下：

```cpp
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

其中 `sockfd` 是客户端建立的 socket，`addr` 参数是服务端的地址，`addrlen` 则是服
务端地址的长度。下面分析 `connect` API 的具体实现。

<!--more-->


图解
====

下图是 `connect` 函数的函数调用关系图，这里之列出重要的函数，其中带有 `*` 表示是
通过函数指针间接引用

![apiconnect][apiconnect]

  [apiconnect]: /img/posts/apiconnect.png

详细说明
========

`connect`  API 的实现可能是目前位置见过最复杂的，因为前面的 API 无论是 socket,
bind, listen 还是 accept 基本生都是只涉及到本机的数据结构设置。而 `connect` API
的实现则涉及到服务器端和客户端的通信，也就是 3 次握手的具体实现。涉及到数据发送
则不可避免的涉及到 IP 层的相关处理，因此内容非常的多。

这篇文章主要讲解 `connect` 的实现逻辑，至于涉及到的 3 次握手的具体实现和 IP 层处
理在后面的文章中再讲解。

哈希表 ------

在 [socket API 实现（二）—— bind 函数][apibind] 一文中我们提到在内核中有一个哈希
结构包含三个不同的哈希表用来索引不同的状态的 sock 结构，这个哈希结构就是
`inet_hashinfo`。它包含的三个哈希表中，`bhash` 用来存放绑定了本地地址的 sock，
`listening_hash` 用来存放处于监听状态 sock 结构，而  `TCP_ESTABLISHED <=
sk->sk_state < TCP_CLOSE` 的 sock 就是存放在最后一个哈希表 `ehash` 中。

在上图的 `inet_hash_connect` 函数有两个功能，第一个就是给 socket 分配端口，这个
我们在 [socket API 实现（二）—— bind 函数][apibind] 一文的 “端口选择” 一节中有提
到具体的方法。第二个功能就是把这个 sock 加入到 ehash 哈希表中。


SYN 包的构建和发送
----------------

`connect` 函数开始了三次握手中的第一次握手，也就是 SYN 包的发送，这个工作是
`tcp_connect` 完成的。它主要完成以下几件事情：

 1. 调用 `alloc_skb_fclone` 分配新的包的空间。任何数据包都是通过 `sk_buff` 结构
    来表示的，通常我们称之为 SKB，`alloc_skb_fclone` 的空间分配其实也是分为两部
    分的：分配 sk_buff 本身的结构体信息，它相当于是这个数据断的元数据（meta）也
    就是数据的描述和控制信息；分配 data 空间，用于存放具体的数据（其实还包括一个
    `skb_shared_info` 的空间，暂时不知道它的作用是什么）。

 2. 初始化各类控制信息。主要是调用 `tcp_init_nondata_skb` 初始化 TCP_SKB_CB 中的
    字段以及在 sock 中的相关字段。

 3. 调用 `__tcp_add_write_queue_tail` 把 SYN 包放入到 sock 的发送队列中去，也就
    是 `sk_write_queue` 队列。

 4. 调用 `tcp_transmit_skb` 把数据发送出去。


等待链接
--------

和 [socket API 实现（四）—— accept 函数][apiaccept] 中提到的 accept 函数一样，其
实 connect 函数需要等待连接完成。因为连接的建立中间涉及到 3 次握手的完成，所以
`connect` 函数并不会在发送完成 SYN 包之后就立即返回，而是调用
`inet_wait_for_connect` 函数进入等待状态。和 [socket API 实现（四）—— accept 函
数][apiaccept] 中提到的 accept 一样，当前进程会进入 `sk->sk_sleep` 这个等待队列
等待连接建立完成，不同的是对于 accept 来说 `sk->sk_sleep` 里面的进程在等待连接的
到来，而对于 connect 来说 `sk->sk_sleep` 中的进程等待连接的完成。对于同一个
socket 来说这两个状态不可能同时出现（不可能在 connect 的同时会 accept），所以使
用同一个等待队列不会出现问题。关于这个等待队列的初始化可以查看 [socket API 实现
（四）—— accept 函数][apiaccept] 一文的等待队列一小节。


 [apibind]: /2014/10/24/socket-bind/ "bind 函数"

 [apiaccept]: /2014/10/29/socket-accept/ "accept 函数"


