---
title: 使用 vim 的 UltiSnips 插件生成符合 google 规范的头文件保护宏
date: 2016-12-03 19:00:09
tags:
 - C/CPP
---

google 的 cpp style guide 中推荐把头文件保护宏定义成如下格式

```cpp
<PROJECT>_<PATH>_<FILE>_H_
```

目前大部分的头文件保护宏生成工具都没有办法按照路径来生成这个宏。在 Vim 的
UltiSnips 插件的默认 Snippet 代码中可以使用 `once` 来生成类似于下面这样的头文件
保护宏：

```
<FILE>_<RANDOM_CHARS>_H
```

这种默认的行为已经能非常好的避免头文件多重包含问题了，但是从美观性和规范性上来说
，google 的标准写法略胜一筹。幸运的是 UltiSnips 允许你定义自己的 snippet。本文介
绍一种通过自己编写 snippet 的方式生成符合 google 规范的头文件保护宏。

<!--more-->

# 思路

总的来说，算法非常简单，能够生成这种保护宏的前提是你必须有一个项目的根目录（我的
实现中使用了git的根目录，因为日常开发中大部分项目都是使用git作为版本控制器），然
后得到当前工作目录和文件名，之后通过字符串的处理得到基于项目路径的头文件保护宏。
总的来说，你需要：

- 项目根目录
- 当前工作目录
- 当前文件名

## 基础知识

要得到这三个名字，单靠 UltiSnips 本身会比较困难，幸运的是 UltiSnips 支持使用
Python 编写 snippet，本文介绍 snippet 使用了 Python 进行编写。关于如何编写
snippet 以及如何在 snippet 中嵌入 Python 代码 [Vimcasts][vimcasts] 中有三个非常
棒的视频，这里不再重复。

- [Meet UltiSnips][cast1]
- [Using Python interpolation in UltiSnips snippets][cast2]
- [Using selected text in UltiSnips snippets][cast3]

[vimcasts]: http://vimcasts.org/
[cast1]: http://vimcasts.org/episodes/meet-ultisnips/
[cast2]: http://vimcasts.org/episodes/ultisnips-python-interpolation/
[cast3]: http://vimcasts.org/episodes/ultisnips-visual-placeholder/

## 获取项目根目录

你可以通过执行 `git rev-parse --show-toplevel` 这条命令得到 git 项目的根目录。在
Python 中我们可以使用 `subprocess` 的 `check_output` 方法执行系统命令。

```python
import subprocess

cmd = 'git rev-parse --show-toplevel'
root = subprocess.check_output(cmd, shell=True)
```

当然如果你不在一个 git 目录中，执行 `git rev-parse --show-toplevel` 会出错，而使
用 `check_output` 会抛 `CalledProcessError` 异常。处理异常后代码如下：


```python
import subprocess

def get_project_root():
    """Find the git root of project."""
    cmd = "git rev-parse --show-toplevel"
    try:
        return subprocess.check_output(cmd, shell=True)
    except subprocess.CalledProcessError:
        return None
```

## 获取当前的工作目录

和上面的方式一样可以通过 subprocess 执行 `pwd` 命令得到结果。

```python
def get_pwd():
    return subprocess.check_output("pwd", shell=True)
```

## subprocess 返回值的处理

需要注意的是，subprocess 返回的并不是 str，而是 `byte string` 为了能够得到
`unicode string` 我们需要对返回值做一个额外的转换。

```python
root = get_project_root()
if root:
    root = root.decode('utf-8').rstrip()
```

上面做了一个额外的 `rstrip` 操作用来移除返回值末尾的 `\n`

## 当前目录相对于项目根目录的路径

```python
path = pwd[len(root) + 1:]
```

## 当前文件名

`UltiSnips` 本身提供了这个名称，所以不需要我们做额外的处理，直接使用 `snip.fn`
就可以了。

## 把路径转换成宏定义

直接把路径中的 `/` 转换成 `_` 之后拼接，转大写即可。

```python
import re

def replace_nonalphnum(token, replacement):
    return re.sub(r'[^A-Za-z0-9]+', replacement, token)

tokens = [replace_nonalphnum(token, '_') for token in (path, snip.fn)]

snip.rv = '_'.join(tokens).upper() + '_'
```

## 完整代码

你可以在我的配置文件[仓库][repo]中找到完整的代码。仓库中的代码，对于找到项目根目
录的情况做了兼容，回退到之前使用随机数后缀的方式。

[repo]:
https://github.com/zhaohuaxishi/dotfiles/blob/master/.vim/UltiSnips/c.snippets

# 参考

- [Ultisnips](https://github.com/SirVer/ultisnips)
- [Convert bytes to a Python string](http://stackoverflow.com/questions/606191/convert-bytes-to-a-python-string)
- [Find the root of the git repository where the file
    lives](http://stackoverflow.com/questions/22081209/find-the-root-of-the-git-repository-where-the-file-lives)
