---
author: 郭荣飞
date: 2015-02-16
title: Netlink 库 -- 官方开发者教程中文版第八部分
tags:
 - netlink
 - libnl
---

<H1 id = "cache_system">8. 高速缓存系统</H1>

<H2 id = "alloc_cache">8.1. 高速缓存的分配</H2>

基本上所有的子系统都提供了一个函数用来分配一个某种形式的高速缓存。这个函数通常
是像这个样子的：

	struct nl_cache *<object name>_alloc_cache(struct nl_sock *sk);

这个函数为自己的对象类型分配一个新的高速缓存、适当的初始化它，然后对它进行更新
以便同步主存储区的当前状态。比如，一个链路高速缓存将会包含目前内核中已经配置的
所有的链路。

有些分配函数还会接收额外的参数以确定高速缓存中应该包含什么数据。

所有的这些函数都会返回一个新分配的高速缓存，或者是在出错的时候返回一个 `NULL`。

<!--more-->

<H2 id = "cache_manager">8.2. 高速缓存管理器</H2>

高速缓存管理器的作用是跟踪缓存区并自动接收事件通知以便保持缓存和内核状态的同步。
每一个高速缓存管理器都只有一个 `neltink` 套接字与之关联，这就把每个管理器限制在
一个 `netlink` 协议簇内。所以所有交给管理器的高速缓存都必须是同一个 `netlink`
协议簇的某一部分。管理器本身的特性决定了一个高速缓存无法同时维护同一个类型的缓存
的两个实例。与之关联的套接字会订阅每一个高速缓存的事件通知组并处于非阻塞模式。
库中存在许多函数不断的对套机字进行轮询（`poll()`）来等待新的事件的到来。

	       App         libnl           Kernel
		|                            |
		    +-----------------+        [ notification, link change ]
		|   |  Cache Manager  |      | [   (IFF_UP | IFF_RUNNING)  ]
		    |                 |                |
		|   |   +------------+|      |         |  [ notification, new addr ]
	    <-------|---| route/link |<-------(async)--+  [  10.0.1.1/32 dev eth1  ]
		|   |   +------------+|      |                      |
		    |   +------------+|                             |
	    <---|---|---| route/addr |<------|-(async)--------------+
		    |   +------------+|
		|   |   +------------+|      |
	    <-------|---| ...        ||
		|   |   +------------+|      |
		    +-----------------+
		|                            |


**创建新的高速缓存管理器**

	struct nl_cache_mngr *mngr;

	// 为 RTNETLINK 分配一个新的高速缓存管理器并且自动提供
	// 添加到管理器中的高速缓存
	err = nl_cache_mngr_alloc(NULL, NETLINK_ROUTE, NL_AUTO_PROVIDE, &mngr);

**高速缓存的跟踪**

	struct nl_cache *cache;

	// 为链路/接口（links/interfaces）分配一个新的高速缓存并要求管理器为我们
	// 维护它的更新。这将会发送一个全新的复制请求以初始化填充这个缓冲区。
	cache = nl_cache_mngr_add(mngr, "route/link");

**使得管理器接收更新**

	// 使得管理器可以接收更新，这将会没 5 秒中调用一个 poll() 函数。
	if (nl_cache_mngr_poll(mngr, 5000) > 0) {
		// 管理器至少收到了一个更新，需要复制高速缓存吗？
		nl_cache_dump(cache, ...);
	}

**释放缓存管理器**

	nl_cache_mngr_free(mngr);

