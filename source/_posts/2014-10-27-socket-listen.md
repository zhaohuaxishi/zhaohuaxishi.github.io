---
author: 郭荣飞
date: 2014-10-27
title: socket API 实现（三）—— listen 函数
tags:
 - Linux Kernel
 - socket
---

概述
====

前两篇文章中，我们已经分析了 socket API 中用来创建 socket 的 `socket` 函数和用来
绑定地址的 `bind` 函数。正常到服务器程序在地址绑定之后就会在这个地址上开始监听等
待链接，这一工作可以调用 `listen` 函数来完成，代码如下：

``` c
listen(server_fd, 10);
```

这篇文章将要分析上面这条语句在内核中的具体实现流程。

<!--more-->


图解
====

和前面的文章一样，我们首先看看函数调用关系的图解。带 `*` 的函数表示是通过函数指
针引用而不是直接调用

![listen 系统调用](/img/posts/apilisten.png)

`listen` 函数到逻辑比 `socket` 和 `bind` 到函数逻辑要简单许多。
`sockfd_lookup_light` 函数在 `bind` 函数到讲解中提到过，这里不再重复。如果你不是
了解其中到机制，可以参考我的另外两篇文章 [**socket API 实现（一）—— socket 函数
**][apisocket] 和 [**socket API 实现（二）—— bind 函数**][apibind]。

这篇文章将会重点讲解 `inet_listen` 函数和 socket 中和监听相关的数据结构。

连接请求队列
===========

连接请求队列分为半连接队列和全链接队列，这两个队列处于同一个结构体中，也就是
`request_sock_quue` `inet_connection_sock` 结构的 `icsk_accept_queue` 字段就是这
个类型，用来存储一个 `sock` 的连接请求。`request_sock_quue` 的结构定义如下：

``` c
struct request_sock_queue {
    struct request_sock	*rskq_accept_head;
    struct request_sock	*rskq_accept_tail;
    rwlock_t		syn_wait_lock;
    u8			rskq_defer_accept;
    /* 3 bytes hole, try to pack */
    struct listen_sock	*listen_opt;
};
```

其中前面两个字段用来全连接队列，而最后一个字段用来存储半连接队列。全连接队列是随
着连接请求不断到来并且被处理之后才逐步建立起来的。半连接队列则在 `listen` 函数中
创建，因为 `listen` 的第二个参数 `backlog` 指定了同时允许的最大半连接状态的连接
数，也就确定了半连接队列的长度，比如 `listen(fd, 10)` 中的 10。

在 `inet_listen` 中调用了 `reqsk_queue_alloc` 函数创建 sock 的半连接队列，也就是
分配和初始化 `listen_sock` 结构空间。代码如下：

``` c
size_t lopt_size = sizeof(struct listen_sock);
struct listen_sock *lopt;

nr_table_entries = min_t(u32, nr_table_entries, sysctl_max_syn_backlog);
nr_table_entries = max_t(u32, nr_table_entries, 8);
nr_table_entries = roundup_pow_of_two(nr_table_entries + 1);
lopt_size += nr_table_entries * sizeof(struct request_sock *);
if (lopt_size > PAGE_SIZE)
    lopt = __vmalloc(lopt_size,
        GFP_KERNEL | __GFP_HIGHMEM | __GFP_ZERO,
        PAGE_KERNEL);
else
    lopt = kzalloc(lopt_size, GFP_KERNEL);
```

这段代码比较的晦涩，主要是因为 `listen_sock` 结构的定义非常的灵活：

```cpp
struct listen_sock {
    ......
    u32			nr_table_entries;
    struct request_sock	*syn_table[0];
};
```

在结构体定义到末尾有一个零长数组，当我们分配的空间大于 `sizeof(struct
listen_sock)` 的时候，`syn_table` 字段会指向超出结构体大小空间的第一个字节。关于
零长数组到更多介绍，可以参考我写的另外一片文章 [**C 语言的趣味用法**][cuse]。

在上面到空间分配代码中，`lopt_size` 首先是 `sizeof(struct listen_sock)` 因为在
`request_sock_queue` 中表示半连接队列的 `listen_opt` 是一个指针，它到空间我们也
许要分配。之后 `lopt_size += nr_table_entries * sizeof(struct request_sock *);`
表示在 `listen_sock` 空间之后还有额外的 `nr_table_entries * sizeof(struct
request_sock *)` 空间，也就是我们前面提到的 `syn_table` 指向的空间。

从这段空间到分配我们可以看出，每一个连接请求都是一个 `requset_sock` 结构，半连接
状态的 `request_sock` 的指针存放在 `syn_table` 中，这种连接请求最都有
`nr_table_entries` 个。它的最大值由下面三个中的最小值决定：参数 `backlog`，
`sysctl_max_syn_backlog` 变量， `sock_net(sock->sk)->core.sysctl_somaxconn`。其
中 `backlog` 和 `somaxconn` 的比较在 `sys_listen` 中就已经完成并把较小的一个作为
参数传递给了 `reqsk_queue_alloc`。剩下两个的比较则在前面的分配代码中完成。

分配空间之后 `sock` 的内存空间结构大致如下：

![request_sock_queue](/img/posts/request_sock_queue.png)


`reqsk_queue_alloc` 函数的主要工作就是分配上图中 `listen_sock` 的结构空间。


端口重用
========

在 `inet_listen_start` 中会调用 `inet_csk_get_port` 函数。这看起来不可思议，因为
端口绑定的工作是在 `bind` 函数中进行的，`inet_csk_get_port` 函数我们在 [**socket
API 实现（二）—— bind 函数**][apibind] 一文中已经提过，这里不再解释它的代码。我
看过的资料大部分都说这里调用 `inet_csk_get_port` 是为了检查是否绑定了端口，是否
端口占用等等。个人认为这些说法都有失偏颇。

其实这里调用 `inet_listen_start` 主要是有两个作用。第一如果没有绑定端口，那么先
进行端口绑定，如果绑定了端口则需要更改 `inet_bind_bucket` 中的 `fastreuse` 字段
，把他设置成否定值。这个字段的作用我们在[**socket API 实现（二）—— bind 函数
**][apibind] 一文中已经解释过，在 `inet_csk_start_listen` 中设置了 `sock_state`
为 `TCP_LISTEN` 状态，这就使得端口可重用的第二个判断条件不成立了，也就是说
`fastreuse` 字段失效。调用 `inet_csk_get_port` 函数可以让可重用再次得到修正。以
下是简化的代码：


```cpp
if (!snum) {
    ......
} else {
    ......
    inet_bind_bucket_for_each(tb, node, &head->chain)
        if (ib_net(tb) == net && tb->port == snum)
            goto tb_found;                                      (1)
}

tb_found:                                                       (2)

if (!hlist_empty(&tb->owners)) {
    if (... && sk->sk_state != TCP_LISTEN ) {
        ......
    } else {
        if (inet_csk(sk)->icsk_af_ops->bind_conflict(sk, tb)) { (3)
            ......
        }
    }
}

if (!tb && ...) == NULL)
    ......
if (hlist_empty(&tb->owners)) {
    ....
} else if (tb->fastreuse &&
       (!sk->sk_reuse || sk->sk_state == TCP_LISTEN))
    tb->fastreuse = 0;                                          (4)
```


如果地址已经绑定，那么最开始的判断 `!sum` 不成立所以执行 `else`（1）。因为地址已
经绑定了所以可以找到 `inet_bind_bucket` 结构，跳到 `tb_found:`（2）。以为已经绑
定地址所以 `tb->owners` 不会为空（绑定的时候调用 `inet_bind_hash` 会把 `sk` 添加
到 `tb->owners` 中），所以接下来的 `if` 中会判断是否地址冲突，如果之后自己在
`owners` 队列中的话不会冲突（3）的判断为假不会执行。所以最后的流程到（4）设置
`fastreuse` 为假。

所以从上面的代码来看， 在 `inet_csk_start_listen` 中调用 `inet_csk_get_port` 主
要作用有三点：

 1. 如果没有绑定端口，先绑定端口

 2. 如果端口已经绑定检查是否有冲突

 3. 如果没有冲突把端口可重用的判断标志设置为否


加入监听哈希表
=============

在 [**socket API 实现（二）—— bind 函数**][apibind] 一文中我们说过，为了快速的找
到 sock 结构，内核设置了三个哈希表，这三个哈希表在同一个结构 `inet_hashinfo` 之
中。和上一篇文章主题——绑定相关的哈希表是 `bhash` 字段，而和这篇文章的主题——监听
相关的哈希表是 `listening_hash` 函数调用关系图中的最后一个函数 `inet_hash` 就是
用来把 `sock` 加入到 `listening_hash` 队列中去的。

```cpp
if (sk->sk_state != TCP_LISTEN) {
    __inet_hash_nolisten(sk);
    return;
}

ilb = &hashinfo->listening_hash[inet_sk_listen_hashfn(sk)];
__sk_nulls_add_node_rcu(sk, &ilb->head);
```

在《追踪 Linux TCP/IP 代码运行》一书的第七章 242 页中分析 `__inet_hash_nolisten`
的运行显然牛头不对马嘴。因为此时 `sk->sk_state == TCP_LISTEN`。这个函数的作用其
实是找打该 sock 对应的哈希桶（ilb），然后把这个 sock 链接到这个哈希桶的哈希链中
。 `struct sock` 中对应的 `sk_nulls_node` 字段被用来链接这个哈希链，但是内核的代
码中说是提供给 `udp/udp-lite` 协议似乎有点误导的嫌疑。

```cpp
*	@skc_nulls_node: main hash linkage for UDP/UDP-Lite protocol
struct sock_common {
    union {
        struct hlist_node	skc_node;
        struct hlist_nulls_node skc_nulls_node;
    };
};

struct sock {
    struct sock_common	__sk_common;
#define sk_nulls_node		__sk_common.skc_nulls_node
};
```

从上面的代码可以看出 `sock` 的 `sk_nulls_node` 来自 `sock_common`，而
`sock_common` 对于 `skc_nulls_node` 的注释说它是提供给 `udp/udp-lite` 协议使用的
。


* * *


[apisocket]: /2014/10/23/socket-create/ "socket API 实现（一）—— socket 函数"

[apibind]: /2014/10/24/socket-bind/ "socket API 实现（二）—— bind 函数"


[cuse]: /2014/09/30/c-language-funny-usage/ "C 语言的趣味用法"
