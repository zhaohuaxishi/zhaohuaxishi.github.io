---
author: 郭荣飞
date: 2014-10-24
title: socket API 实现（二）—— bind 函数
tags:
 - Linux Kernel
 - socket
---

**注：本文基于 2.6.32 版本的内核**


概述
====


[**上一篇**][apisocket] 文章中说明了使用 `socket()` 函数创建一个 socket 在内核中
的处理流程。再经过一些来到处理之后，返回 socket 的文件描述符。通常应用程序下一步
调用的函数就是 `bind()`
比如：

```cpp
bind(server_fd, (struct sockaddr*)&server_address, server_len);
```

那么这一条语句在内核中到处理又是如何呢？我们将在这篇博文中进行分析。和上一篇文章
一样，这篇文章不打算过多的分析代码，而是从总体结构上给出一个较为宏观的图解分析。


<!--more-->


图示
====

`bind()` 函数最终在内核中的函数调用关系如下图所示：

![bind 函数调用关系][apibind]

  [apibind]: /img/posts/apibind.png

**注：上图中带星号的函数表示是通过函数指针间接引用的**

`sys_bind` 调用的前两个函数逻辑非常到简单，如何从文件描述符（fd）得到 socket 结
构我在 [**socket API 实现（一）—— socket 函数**][apisocket] 文章中已经提过，这里
不再赘述。而第二个 `move_addr_to_kernel` 只是对 `copy_form_user` 的一次封装。

`inet_bind` 的函数关系较为复杂，我们在下一小节中分析。


端口绑定
========

一般设置端口分为两种，指定端口和不指定端口。如果指定了端口，则尝试使用这个端口，
如果没有指定则随机分配一个端口。无论是那一种绑定，我们都必须知道这个端口是否能用
。

那么系统如何知道那些端口能用，那些端口不能用呢？这一切都靠一个哈希表的存在——
`inet_hashinfo`。对于一个 tcp socket 来说，它是定义在 net/ipv4/tcp_ipv4.c 中的全
局变量 `tcp_hashinfo`。

哈希表
------

`inet_hashinfo` 里面并不是只包涵一个哈希表，而是三个哈希表的集合。分别是：


  1. ehash - 用来索引 `TCP_ESTABLISHED <= sk->sk_state < TCP_CLOSE` 的 sock

  2. bhash - 用来索引绑定本地地址的 sock

  3. `listening_hash` - 用来索引 `sk->sk_state` 为 `TCP_LISTEN` 状态的 sock


在这三个哈希表中，前面两个是动态分配的，后面一个则是静态分配到数组。它们的创建和
初始化是在 `tcp_init()` 函数中完成的，而这个函数会在 `inet_init()` 中调用，后者
则是通过 `fs_initcall(inet_init);` 调用在编译期间注册到初始化端中，所以会在内核
启动到时候执行。

和我们本文主题相关的就是第二个哈希表 bhash。 `inet_hashinfo` 中和它相关的字段有
三个：

```cpp
struct inet_bind_hashbucket	*bhash;
unsigned int			bhash_size;
struct kmem_cache		*bind_bucket_cachep;
```

第一个字段定义哈希表，第二个字段记录哈希表的长度，而第三个字段是一个 SLAB 高速缓
存，作用我们在后面再说。从第一个字段的定义可以知道整个哈希表是一个指针数组。数组
的每一个元素都是一个 `inet_bind_hashbuchet` 指针。 `inet_bind_hashbucket` 的定义
如下：

```cpp
struct inet_bind_hashbucket {
    spinlock_t		lock;
    struct hlist_head	chain;
};
```

`chain` 字段用来链接具有同一哈希值的哈希元素，也就是 `inet_bind_bucket`。它的定
义如下。

```cpp
struct inet_bind_bucket {
#ifdef CONFIG_NET_NS
    struct net		*ib_net;
#endif
    unsigned short		port;
    signed short		fastreuse;
    int			num_owners;
    struct hlist_node	node;
    struct hlist_head	owners;
};
```

他们之间通过内部的 `node` 字段链接起来。而 `inet_bind_habuchket` 中的 `lock` 字
段则是用来同步 `chain` 字段的访问。

`inet_bind_bucket` 结构就是用来描述端口和 sock 之间的绑定关系的。它的 `port` 字
段表示一个绑定的端口，而 owners 则表示绑定到这个端口之上的所有 sock，因为端口可
以重用，所以同一端口可能有多个 sock 绑定。

所以这个 `bhash` 哈希表的结构大概如下：

![bhash 结构][bhash_struct]

  [bhash_struct]: /img/posts/bhash_struct.png

这张图我们在后续的讲解中会继续用到。

端口的选择
----------

我们前面提到，端口绑定分为两种，一个中指定端口，一种者随机选择。如果给 `bind` 传
递的地址参数中，port 字段为 0，那么就会自动选择参数。步骤如下：

  1. 使用 `inet_get_local_port_range` 取得端口的可用区间（目前是 [32768, 61000]
     ），随机选择区间内部的一个端口。

  2. 找到端口对应的 `inet_bind_hashbucket`，遍历它的 chain，如果 chain 中没有元
     素或者有元素可以重用，把这个随机端口作候选端口为跳转到第 4 步。否则跳转到第
     3 步。

  3. 递增随机端口，如果随机端口超过上界，返回下界重新开始。同时递减可用端口数量
     ，如果可用端口数量为 0 失败返回。否则执行第 2 步。

  4. 如果还没有 `inet_bind_bucket` 那么先调用 `inet_bind_bucket_creat` 新建一个
     结构，新建使用到的缓存就是前面提到的 inet_hashinfo 中的 bind_bucket_cachep
     。

  5. 调用 `inet_bind_hash` 绑定端口，也就是设置 `inet_bind_bucket` 中 port 字段
     ，以及 inet_sk(sk) 中的 num 字段，然后把 sk 加入到 `inet_bind_bucket` 的
     owners 队列中。结构可以参考上图。

如果给定了端口，那么直接从上面的第 4 步开始执行。只不过如果 `inet_bind_bucket`
结构如果已经存在，那么需要检查是否可重用，不可重用的话表示端口冲突，调用
`inet_sk(sk)` 的 `icsk_af_ops` 字段中的 bind_conflick 函数处理冲突。这个函数在
sock 建立的时候被初始化为 `inet_csk_bind_conflict`。关于 sock 的创建可以参考我的
上一篇文章 [socket API 实现（一）—— socket 函数][apisocket]。

端口复用
--------

关于什么时候能够端口复用，在 /net/ipv4/inet_hashtable.c 中有详细的注释，下面这一
段是从注释中截取下来的。


    1) Sockets bound to different interfaces may share a local port.
       Failing that, goto test 2.

    2) If all sockets have sk->sk_reuse set, and none of them are in
       TCP_LISTEN state, the port may be shared.
       Failing that, goto test 3.

    3) If all sockets are bound to a specific inet_sk(sk)->rcv_saddr local
       address, and none of them are the same, the port may be shared.
       Failing this, the port cannot be shared.

关于第二点，内核做了优化，为了避免每次都遍历 `inet_bind_bucket` 的 `owners` 字段
来获知是否所有的 sock 都设置了 `sk_reuse` 字段，并且不是在 `TCP_LISTEN` 状态。在
`inet_bind_bucket` 结构体中设置了 `fastreuse` 字段。如果 owners 没有元素，那么这
个字段为真。此后每次添加一个新的 sock 到 owners 中的时候，如果它设置了 sk_reuse
并且不在 `TCP_LISTEN` 状态，就维持 `fastreuse` 为真，否则设置它为假。

所以测试一个端口是否可以复用，只需要测试i `fastreuse` 字段就可以判断上面的第二点
了。


总结
====


经过前面的处理之后，在一个 tcp socket 的 `inet_sock` 中，设置了以下这些字段：

```cpp
inet->rev_saddr = inet->saddr = addr->sin_addr.s_addr (传递进来的 IP)
inet->num = snum (传递进来的端口)
inet->sport = htons(inet->num) (转换了字节序的端口)
inet->daddr = 0;
inet->dport = 0;
```

**注意： 在一个 tcp_sock 中包含 inet_sock。他们之间的关系请参考我的[上一篇文章
][apisocket] 中关于 sock,inet_sock,inet_connetion_sock,tcp_scok 关系的讨论**


  [apisocket]: /2014/10/23/socket-create "socket API 实现（一） —— socket 函数 "

