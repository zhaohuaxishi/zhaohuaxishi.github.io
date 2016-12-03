---
author: 郭荣飞
date: 2014-10-29
title: socket API 实现（四）—— accept 函数
tags:
 - Linux Kernel
 - socket
---

**注：本文基于 2.6.32 版本的内核**


概述
====


正常的 socket 编程函数调用顺序一般是 `socket` -> `bind` -> `listen` ->
**`accept`**。我们在前面的文章中分析了前面三个函数，现在我们分析第四个函数
`accept`。

```cpp
accept(server_fd, (struct sockaddr *))&client_addr, client_len);
```

这个函数在 socket 开始监听之后调用，如果成功接受链接请求，则返回新的客户端
socket 文件描述符，并把客户端的地址放入到 `client_addr` 表示的地址中去。

我们将在这篇博文中详细的讲解 `accept` 函数在内核内部的处理流程。

<!--more-->

图解
====

按照惯例，我们首先给出函数调用关系的图解，带 * 表示这个函数是通过函数指针简介引用：

![accept 系统调用][apiaccepttop]
![accept 系统调用][apiacceptdown]

  [apiaccepttop]: /img/posts/apiaccepttop.png
  [apiacceptdown]: /img/posts/apiacceptdown.png

重点函数说明
===========

`accept` 实际使用到系统调用是 `sys_accept4`，它首先创建一个新的 socket 结构（
`sock_alloc`）以及 file 结构和新的文件描述符（`sock_alloc_fd`），之后设置 socket
结构和 file 结构的相互引用（`sock_attach_fd`）。在函数的最后调用 `fd_install` 使
得 fd 绑定 file 结构。这些函数的功能在 [**socket API 实现（一）—— socket 函数
**][apisock] 一文中就已经讲解过了。这里不在重复。

`sys_accept4` 最主要的操作是调用 `inet_accept` 进而调用 `inet_csk_accept` 等待连
接的到来，然后返回新连接的 sock 结构。

等待连接的到来这个操作是通过调用 `inet_csk_wait_for_connect` 这个函数来完成的，
它使用到的工具就是 “wait queue”。

等待队列
--------

“wait queue” 在实现中主要涉及两个结构。`wait_queue_head_t` 用来代表一个等待队列
以及 `wait_queue_t` 用来代表一个等待的进程。

在每一个 sock 结构中都有一个 `wait_queue_head_t` 类型的字段 `sk_sleep` 用来表示
在这个 `sock` 上等待事件发生。比如对于服务器的 sock 来说，它等待连接的到来。

sock 中的 `sk_sleep` 在 `sock_init_data` 函数（[**socket API 实现（一）—— socket
函数**][apisock]）中有如下初始化：

```cpp
sk->sk_sleep	=	&sock->wait;
```

也就是说，其实 sock 结构的等待队列其实就是 socket 结构的 `wait` 字段。而 socket
结构的 `wait` 字段在 socket 的创建过程函数 `sock_alloc_inode` 中初始化的：

```cpp
static struct inode *sock_alloc_inode(struct super_block *sb)
{
    struct socket_alloc *ei;

    ei = kmem_cache_alloc(sock_inode_cachep, GFP_KERNEL);
    if (!ei)
        return NULL;
    init_waitqueue_head(&ei->socket.wait);   <--

    return &ei->vfs_inode;
}
```

在 `inet_csk_wait_for_connect` 函数中，调用了 `prepare_to_wait_exclusive` 把当前
的进程加入到 sock 的 sleep 队列中。然后使用 `schedule_timeout` 来设置超时定时器
。

在系统中会有定时的时间中断，定时器的原来就是通过记录中断次数知道已经过去来多长时
间。关于定时器到更多讲解请参考 《Linux Kernel Development》第三版中的第十一章。

`schedule_timeout` 最终会在设置定时器之后调用 `schedule` 函数选择其他的进程运行
，而当前的进程则会进入睡眠。这也是为什么我在函数调用图的中间插入了一条分割线的原
因。

连接队列
--------

我们在 [**socket API 实现（三）—— listen 函数**][apilisten] 一文中有提到过，
listen 会创建 sock 的连接队列，它分为全连接队列和半连接队列。当连接建立好了之后
半连接状态到 `request_sock` 就会被移到全连接队列中。对于 `tcp_sock` 来说，这个连
接队列是 `inet_connection_sock` 的 `icsk_accept_queue` 字段，它是
`request_sock_queue` 类型，其定义如下：

```cpp
struct request_sock_queue {
    struct request_sock	*rskq_accept_head;
    struct request_sock	*rskq_accept_tail;
    ......
    struct listen_sock	*listen_opt;
};
```

`rskq_accept_head` 就是全连接队列到头部，而 `listen_opt` 是管理半连接状态请求的
结构。如果连接建立成功，一个连接请求将会从 `listen_opt` 的相关字段中移到
`rskq_accept_head` 指向的队列中。

在 accept 的函数调用图中的 `reqsk_queue_get_child` 函数就是在有连接到来（
`rskq_accept_head` 队列不空）之后从全连接队列中取出一个连接请求 `request_sock`，
其定义如下：

```cpp
struct request_sock {
    struct request_sock		*dl_next; /* Must be first member! */
    ......
    struct sock			*sk;
    ......
};
```

这个结构体中的 dl_next 字段用来链接所有的请求，而代表单个请求的 sock 结构放在
`sk` 字段中。这个字段最终会被当成返回值传递给 `inet_accept` 函数，并通过
`sock_graft` 函数设置好新的 socket 和新的 sock 结构之间到相互引用关系。

地址设置
--------

最终函数返回到 `sys_accept4` 中，调用 `inet_getname` 得到地址，其实也就是读取
sock 结构中的相应值而已。读取完地址之后通过 `move_addr_to_user` 把地址返回到用户
空间。

总结
====

其实 `accept` 的作用就是有新的客户端连接到来的时候创建一个新的 socket 而已。
`sys_accept4` 首先创建好 `socket` 结构，`file` 结构，并分配好文件描述符。之后获
取 `socket` 结构对应的 `sock` 结构。形成一个完整的 socket 结构之后把 socket 文件
描述符返回用户空间，这样服务器就可以通过这个新的 socket 和客户端进行通信了。


 [apisock]: http://blog.guorongfei.com//linux%20kernel/socket/2014/10/23/socket-create.html
            "socket 函数 "

 [apilisten]: http://blog.guorongfei.com/linux%20kernel/socket/2014/10/27/socket-listen.html
            "listen 函数"
