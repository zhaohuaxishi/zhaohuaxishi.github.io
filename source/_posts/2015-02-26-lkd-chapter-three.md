---
author: 郭荣飞
date: 2015-02-26
title: LKD 总结 —— 第三章
tags:
 - Linux Kernel
 - LKD
---

# 概述#

这一章主要介绍的是进程的生命周期，包括进程如何表示，如何创建，如何销毁等等。第一
遍看这些内容的时候觉得有很多的地方都不太理解，重温一遍结合源代码理解之后有了更多
的领悟。文中引用的所有的源代码都是 `linux-2.6.32.65` 内核版本。

<!--more-->

# 进程描述符#

在内核的内部，一个进程是由它的描述符来标识的，它包含了进程的所有信息。在其他的
操作系统教程中通常也会把它称为进程控制块（PCB）。Linux 内核的内部是通过
`struct task_struct` 结构体来表示进程描述符的，这是一个非常庞大而复杂的结构体。
书中简单的列举了它的一些这一章和下一章用到的重要字段：

	struct task_struct {
		unsigned long state;
		int prio;
		unsigned long policy;
		struct task_struct *parent;
		struct list_head tasks;
		pid_t pid;

		...

	};

从上面的结构体定义可以看出，在 Linux 中 `process` 其实和 `task` 应该是等价的。
此外因为 Linux 没有单独的结构体来表示线程，所以 `thread` 在内核的内部和
`process` 并没有什么本质的区别，也就和 `task` 没有本质的区别。

## 进程列表##

书中在这一章中多次提到了进程列表（task list）这个概念，但是书中并没有具体的介绍
它到底是什么东西，下面是我个人的理解：

要理解进程队列的概念首先还要理解内核链表的实现方式。

### 内核链表###

大部分教科书上提到的链表实现方式是把一个结构体变成链表，比如：

	struct node {
		int 		data;
	};

改造成：

	struct node {
		int 		data;
		struct node 	*prev;
		struct node 	*next;
	}

而内核对于链表的实现方式是把链表的节点嵌入到结构体中去。所谓的链表节点就是两个
指向不同方向的指针。

	struct list_head {
		struct list_head *prev;
		struct list_head *next;
	}

如果想要把数据连接起来，只要在结构体中加入一个链表节点就可以了：

	struct node {
		int 		data;
		struct list_head *list;
	}

乍一看两者没有区别，后面这种方式似乎还更加的复杂（node->prev 现在要通过
node->list->prev 才能访问）。但是仔细想想就会发现，这种做法有着极大的灵活性，因为
它**实现了链表结构部分和数据部分解耦**。

这个最直接的好处是，如果我需要把数据加入到一个新的链表中去的话，我只需要加入一个
新的链表节点。

	struct node {
		int 		data;
		struct list_head *list1;
		struct list_head *list2;
	};

如果不用这种方式你可能需要写下面这样的代码：

	struct node {
		int data;
		struct node *prev1;
		struct node *next1;
		struct node *prev2;
		struct node *next2;
	};

如果你想要把节点加入到第三个、第四个、第五个 ...... 链表中。

这样做还有另外一个好处就是，理论上你可以用不同的数据结构来形成一个链表。

	struct foo {
		int foo1;
		int foo2;
		struct list_head *list;
	} foo_instance;

	struct bar {
		int bar1;
		int bar2;
		struct list_head *list;
	} bar_instance;

当然这种做法本身是一种非常糟糕的方式。

### 进程列表的实现###

在 `tasks_struct` 结构体中有一个字段 `tasks` 它的类型是 `list_head`，它相当于是
双向链表的一个节点。通过这个节点，每一个 `task_struct` 结构被链接在了进程列表中。
最终也就得到书中提到的进程列表。这一点可以在后面提到的 `for_each_process()` 函
数中体现。

## 进程描述符的分配##

在 2.6 内核之前 `struct task_struct` 结构体是固定在每个进程的内核栈的底部的，也
就是说分配一个内核栈就等于分配了一个进程描述符。这样就可以非常容易的获得这个
描述符，因为栈的指针存放在寄存器中。

	+--------------------+      --- 高地址
	|                    |       ^
	|                    |       |
	|                    |    内核栈
	|--------------------|       |
	| struct task_struct |       v
	+--------------------+      --- 低地址

现在 `task_struct` 是由 `slab allocator` 分配，也就是说 `struct task_struct`
结构体的地址并不固定，为了能够方便的获取进程描述符，在栈的底部存放了一个新的
结构体 `struct thread_info` 而这个结构体中存在一个指向 `slab allocator` 分配的
`struct task_struct` 结构体的指针。

					+--------------------+      --- 高地址
	+--------------------+		|                    |       ^
	| struct task_struct |		|                    |       |
	+--------------------+		|                    |    内核栈
		^			|--------------------|       |
		+----------------------	| struct thread_info |       v
					+--------------------+      --- 低地址

关于这两个结构体之间的相互引用关系可以查看后面关于新建进程描述符中的解释。

# 进程的标识#

对于应用程序来说，我们可以调用 `getpid()` 系统调用来得到一个进程的标识。而在内核
的内部，唯一确定一个进程的就是它的进程描述符，它可以通过调用 `current` 宏来实现。

因为 `current` 宏返回的是当前进程的进程描述符，所以这个宏只有在 *进程上下文* 中
才是有效的的，因为与子对应的 *中断上下文* 没有相关的进程存在，也就没有进程描述符。

# 进程树#

在 Linux 内核中，进程被组织成了一颗树的形式，所有的进程都是 `init` 进程的后代。
在 `task_struct` 结构体中存在一个指向父进程的 `task_struct` 的字段 `parent`[^1]
以及一个存放子进程的 `children` 字段。

书中的遍历进程的子进程的例子如下：

	list_for_each(list, &current->children) {
		task = list_entry(list, struct task_struct, sibling);
	}


上面这个例子中比较奇怪的一点是 `list_entry` 函数使用的 `sibling` 字段而不是
`children` 字段。这里之所以写成 `sibling` 的原因是因为其实父进程的所有子进程是
通过子进程的 `sibling` 链接节点链接起来的，而父进程中的 `children` 字段是这个链表
的表头。在内核源码的 `copy_process()` 函数中创建完新的子进程进程描述符之后有下面
这段代码：

	if (likely(p->pid)) {
		list_add_tail(&p->sibling, &p->real_parent->children);
		...
	}

也就是说 `sibling` 字段最终链接到的是 `real_parent->children` 链表。

但是因为 `list_entry` 其实就是 `container_of` 宏的一个别名，而 `container_of`
宏本身并不在乎使用的是哪一个字段。所以如果我的理解没有错误的话， 上面例子中的
`sibling` 可以替换成任何一个 `task_struct` 中的字段。

	#define list_entry(ptr, type, member) \
		container_of(ptr, type, member)

	#define container_of(ptr, type, member) ({                      \
	        const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
		(type *)( (char *)__mptr - offsetof(type,member) );})

在上一个小节中有提到，系统中存在一个进程列表连接所有的进程，这个列表是一个双向的
循环链表。遍历这个链表的方法是调用 `for_each_process`

	#define next_task(p) \
		list_entry_rcu((p)->tasks.next, struct task_struct, tasks)

	#define for_each_process(p) \
	        for (p = &init_task ; (p = next_task(p)) != &init_task ; )

上面的 `next_task` 的实现也进一步证明 `tasks` 字段是用来链接整个进程列表的。

# 进程的创建#

在 Linux 内核中创建新进程通常是通过 `fork()` 系统调用实现的，它实际上则是调用
`clone()` 系统调用，进而调用 `do_fork()` 函数来完成这一工作。它实际上的工作就是
调用 `copy_process()` 复制父进程的进程描述符。`copy_process()` 函数主要完成以下
工作。

  1. 调用 `dup_task_struct()` 创建新的内核栈和以及 `thread_info` `task_struct`
     结构体。

     * 调用 `alloc_task_struct()` 函数使用 `slab allocator` 分配一个新的
       `struct task_struct` 结构。

     * 调用 `alloc_thread_info()` 获得一个新的 `thread_info` 结构体。这个函数在
       `x86` 架构中的实现其实是调用 `__get_free_pages()` 函数。也就是说其实这个
       函数分配了一页作为栈，而把这一页的开头强制转换成 `struct thread_info`。这
       一点可以从后面的赋值语句：`tsk->stack = ti;` 中得知。这也可以解释为什了
       `struct thread_info` 在内核栈的底部，因为内核栈是向下生长，所以页的开头
       位置也就是栈的底部。

     * 调用 `arch_dup_task_struct()` 函数复制原来的进程的 `task_struct` 结构体，
       调用 `setup_thread_stack()` 函数复制原来的 `struct thread_info` 中的数据，
       并设置新的 `thread_info` 的 task 字段，让他指向刚刚分配的 `task_struct`
       结构。这样就形成了进程描述符的分配一节中提到的结构。

  2. 调用 `copy_semundo()` `copy_files()` `copy_fs()` `copy_mm` 等等 `copy_**()`
     函数拷贝相关的进程资源。

  3. 调用 `alloc_pid()` 分配新的 `PID`。从随后的两个赋值语句 `p->pid = pid_nr(pid);`
     和 `p->tgid = p->pid;` 可以看出对于一个普通的进程，它的 `tgid` 和 `pid` 是同一个
     值。

# Linux 中线程的实现方式

在 Linux 没有单独的结构体用来表示 `thread` 这个概念，线程和进程没有本质的区别，
线程只不过是和其他线程共享资源的进程而已。

起 `Solaris` 和 `Mircrosoft Windows` 这些系统中，线程抽象出来只是为了能够实现轻量
级别的调度单元，而在 `Linux` 中线程是为了共享资源而存在因为 `Linux` 的线程本身就
已经足够轻量了，这是这两种操作系统在设计理念的区别。

因为在 Linux 中线程和进程本质上没有区别，都是使用 `task_struct` 来表示的，所以这
其中存在一个非常有趣的问题，那就是 `PID` 的问题。因为同一个进程的所有线程都应该
返回同一个 `PID` 而所有的线程都有自己独立的 `task_struct` 结构也就有不同的 `pid`
字段，也就是线程的 `tid`。

在 `Linux` 的实现中，`task_struct` 结构体中除了存在一个 `pid` 字段之外还存在一个
`tgid` 字段，也就是线程组的概念。从上一小节的第三点中我们知道，当一个进程创建为
普通的进程的时候，`pid` 和 `tgid` 属于同一个值，也就是说它属于一个只包含它自己的
线程组。但是从一个进程派生一个线程（比如通过 `pthread_create()` 函数）的时候，
新产生的 `task_struct` 会分配到一个新的 `pid`，但是它的 `tgid` 和它的父进程保持
一致，这样一来子进程（线程）就加入到了父进程的线程组中。

在 `copy_process()` 函数中有下面代码进一步的证明了上面这一点：

	if (clone_flags & CLONE_THREAD)
		p->tgid = current->tgid;

所以当我们调用 `getpid()` 系统调用的时候，无论是进程还是线程都会放回 `tgid` ，
而调用 `gettid()` 的时候返回的才是 `pid` 字段。下面的代码是内核中这两个系统调用
的实现方式：

	/* Thread ID - the internal kernel "pid" */
	SYSCALL_DEFINE0(gettid)
	{
		return task_pid_vnr(current);
	}

	/**
	 * sys_getpid - return the thread group id of the current process
	 *
	 * Note, despite the name, this returns the tgid not the pid.  The tgid and
	 * the pid are identical unless CLONE_THREAD was specified on clone() in
	 * which case the tgid is the same in all threads of the same group.
	 *
	 * This is SMP safe as current->tgid does not change.
	 */
	SYSCALL_DEFINE0(getpid)
	{
		return task_tgid_vnr(current);
	}


所以其实用户看到的 PID 和内核看到的 PID 并不是同一个值，在内核中使用 `task_struct`
结构体的 `pid` 字段而用户空间看到的则是 `tgid`。

新建进程（线程）的时候这两个值的赋值情况如下图：


				USER VIEW
	 <-- PID 43 --> <----------------- PID 42 ----------------->
			     +---------+
			     | process |
			    _| pid=42  |_
			  _/ | tgid=42 | \_ (new thread) _
	       _ (fork) _/   +---------+                  \
	      /                                        +---------+
	+---------+                                    | process |
	| process |                                    | pid=44  |
	| pid=43  |                                    | tgid=42 |
	| tgid=43 |                                    +---------+
	+---------+
	 <-- PID 43 --> <--------- PID 42 --------> <--- PID 44 --->
				KERNEL VIEW

# 进程的终止#

进程的终止调用的是 `exit()` 系统调用，而这个系统调用实际上调用的是 `do_exit()`
函数：

	SYSCALL_DEFINE1(exit, int, error_code)
	{
	        do_exit((error_code&0xff)<<8);
	}

而 `do_exit()` 函数的框架如下：

	NORET_TYPE void do_exit(long code)
	{
		...
		exit_signals(tsk);  /* sets PF_EXITING */
		...
		tsk->exit_code = code;
		...
		exit_mm(tsk);
		...
		exit_sem(tsk);
		exit_files(tsk);
		exit_fs(tsk);
		exit_thread();
		...
		exit_notify(tsk, group_dead);

		tsk->state = TASK_DEAD;
		schedule();
	}


这个函数最后会调用 `schedule()` 函数选择另一个进程运行，所以这个函数永远都不会
返回。在这个函数中比较值得关注的函数就是 `exit_notify()` 。这个函数会把当前进程
的所有子进程进行 `reparent` 并把当前进程的 `task_struct` 的 `exit_state`
设置成 `EXIT_DEAD` 或者是 `EXIT_ZOMBIE`。它的简化版本如下：

	static void exit_notify(struct task_struct *tsk, int group_dead)
	{
		...
		forget_original_parent(tsk);
		...
		tsk->exit_state = signal == DEATH_REAP ? EXIT_DEAD : EXIT_ZOMBIE;
		...
	}

`do_exit()` 函数并不会调用 `free_task()` 函数来释放进程描述符占用的空间，因为它
需要保留这些信息以便父进程调用 `wait4()` 来获取子进程的终止信息。

书中关于相关函数的介绍并不是非常的清晰，从源码的分析来看，这些函数有以下的调用
关系：`sys_wait4()->do_wait()->do_wait_thread()->wait_consider_task()->
wait_task_continued()->put_task_struct()->__put_task_struct()->free_task()`

最终的 `free_task()` 会调用 `free_thread_info()` 和 `free_task_struct` 释放相关
结构体占用的空间。

# reparent#

如果父进程在子进程之前结束了，那么它永远不会调用 `wait4()` 来获取子进程的终止信息
这样以来就会导致子进程的进程描述符得不到释放，浪费空间。为了解决这个问题，在进程
结束的时候需要对为它所有的子进程设置新的父进程，这一操作叫做 `reparent`。

`exit_notify()` 函数调用的 `forget_original_parent()` 会调用 `find_new_reaper()`
函数，这个函数查找该线程组中的其他进程作为 `reaper` 如果不存在这样的进程就使用
`init` 进程。

涉及到设置新的父进程的时候有另外一个地方比较难以理解，那就是在 `task_struct` 结
构体中存在两个不同的字段表示父进程 `parent` 和 `real_parent` 这两个字段的相关解释
可以参考 <http://lists.kernelnewbies.org/pipermail/kernelnewbies/2012-February/004794.html>
从该文中的解释来看，这个 `real_parent` 才是真正的指向父进程的进程描述符，而
`parent` 则是指向 `ptrace` 的进程描述符。

在 `do_fork()` 创建新的进程的时候，新进程的进程描述符有如下的设置：

	/* CLONE_PARENT re-uses the old parent */
	if (clone_flags & (CLONE_PARENT|CLONE_THREAD)) {
		p->real_parent = current->real_parent;
		p->parent_exec_id = current->parent_exec_id;
	} else {
		p->real_parent = current;
		p->parent_exec_id = current->self_exec_id;
	}

这里都只是涉及到 `real_parent` 的设置。而在 `__ptrace_link()` 函数中有下面的设
置：

	child->parent = new_parent;

所以从这一点来看，书中在 *The Process Family tree* 一节中提到 `task_struct` 中的
`parent` 字段指向父进程的说法并不是特别妥当。
