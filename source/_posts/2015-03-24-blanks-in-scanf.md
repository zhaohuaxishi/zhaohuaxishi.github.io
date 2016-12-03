---
author: 郭荣飞
date: 2015-03-24
title: 关于 scanf() 函数格式字符串中的空白字符
catigories:
 - C
---

# 问题#

最近重读 《The C Programing Language》 一书，7.4 节讲解格式化输入`scanf`，书中
关于 `scanf()` 函数的第一个参数——格式字符串的组成部分描述时有下面这样一句话。

> Blanks or tabs, which are ignored.

然而使用 `man 3 scanf` 查看关于 `scanf()` 函数的文档时，同样关于格式字符串的
组成部分描述有下面这段话：

> A sequence of white-space characters (space, tab, newline, etc.; see
> isspace(3)).  This directive matches  any  amount of white space, including
> none, in the input.

K&R 中说空白是被忽略的成分，而文档中给出的说法是空白会匹配输入流中任意空白字符
（包括 0 ）。那么到底那一种说法比较妥当呢？

<!--more-->

# 例子#

下面写两个例子比较一下：

```cpp
	#include <stdio.h>

	int main(void)
	{
		int day = 0, year = 0;
		char monthname[20] = {0};
		int sretval;

		sretval = scanf("Date: %d %s %d", &day, monthname, &year);

		printf("%d, %d, %s, %d\n", sretval, day, monthname, year);
		return 0;
	}
```

这个程序输入：`Date:28Dec 1998`，输出的结果是：

	3, 28, Dec, 1998

但是如果输入：<code>&nbsp;&nbsp;&nbsp;Date:28Dec 1998</code>，输出的结果是：

	0, 0, , 0

注意第二个输入中在 `Date:` 之前有空格。

如果把上面的例子改写一下：

```cpp
	#include <stdio.h>

	int main(void)
	{
		int day = 0, year = 0;
		char monthname[20] = {0};
		int sretval;

		sretval = scanf(" Date: %d %s %d", &day, monthname, &year); /* 加空格*/

		printf("%d, %d, %s, %d\n", sretval, day, monthname, year);
		return 0;
	}
```

此时输入 `Date:28Dec 1998` 和输入 <code>&nbsp;&nbsp;&nbsp;Date:28Dec 1998</code> 得到的结果都是：

	3, 28, Dec, 1998

从上面这两个例子来看，`man` 文档中的说法好像是妥当一些。
