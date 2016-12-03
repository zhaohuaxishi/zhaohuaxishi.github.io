title: Python 正则表达式疑难点
date: 2016-02-26 21:23:51
tags:
 - Python
---

# `^` 和 `\A` 以及 `$` 和 `\Z` 的区别

Python 的文档中对于这几个的记号的含义解释如下：

- `^`

    > Matches the start of the string, and in MULTILINE mode also matches
    > immediately after each newline.

- `$`

    > Matches the end of the string or just before the newline at the end of
    > the string, and in MULTILINE mode also matches before a newline. foo
    > matches both ‘foo’ and ‘foobar’, while the regular expression foo$
    > matches only ‘ foo’. More interestingly, searching for foo.$ in
    > 'foo1\nfoo2\n' matches ‘ foo2’ normally, but ‘foo1’ in MULTILINE mode;
    > searching for a single $ in 'foo\n' will find two (empty) matches: one
    > just before the newline, and one at the end of the string.

- `\A`

    > Matches only at the start of the string.

- `\Z`

    > Matches only at the end of the string.

从上面的文档看 `$` 和 `\Z` 的含义只有在 MULTILINE 模式下才会有区别，比如说下面
的例子[^1]:

[^1]: 例子来自 [stackoverflow](http://stackoverflow.com/questions/22519318/regex-differences-between-and-a-z)

<!--more-->

```cpp
>>> re.search(r'^word', 'Line one\nword on line two\n', flags=re.M)
<_sre.SRE_Match object; span=(9, 13), match='word'>
>>> re.search(r'\Aword', 'Line one\nword on line two\n', flags=re.M) is None
True
```

```cpp
>>> re.search(r'word$', 'Line one word\nLine two\n', flags=re.M)
<_sre.SRE_Match object; span=(9, 13), match='word'>
>>> re.search(r'word\Z', 'Line one word\nLine two\n', flags=re.M) is None
True
```

# `\number` 的含义

**说来惭愧，这里这个记号其实在大部分语言的的正则表达式中都存在，孤陋寡闻的我以
为只有 Python 中存在这种语法。（2016-03-21）**

<del>这个记号我以前没有在其他语言的正则表达式中见过</del>，它表示前面匹配的第
number个组的内容，Python 文档中给出的含义如下：

> Matches the contents of the group of the same number. Groups are numbered
> starting from 1. For example, (.+) \1 matches 'the the' or '55 55', but not
> 'thethe' (note the space after the group). This special sequence can only be
> used to match one of the first 99 groups. If the first digit of number is 0,
> or number is 3 octal digits long, it will not be interpreted as a group
> match, but as the character with octal value number. Inside the '[' and ']'
> of a character class, all numeric escapes are treated as characters.

主要的点有下面几个：

- 匹配 `number` 组中的内容
- 组号从 1 开始
- 只能匹配前面99组之中的组
- 如果 `number` 以 0 打头或者是三位的 8 进制值，会被当成普通字符
- 如果在 `[]` 之间会被当成普通字符

比如说下面的例子

```cpp
>>> re.match(r'(.+) \1', '555 555') is not None
True
>>> re.match(r'(.+) \01', '555 \01') is not None
True
>>> re.match(r'(.+) [\1]', '555 \1') is not None
True
>>>
```

这个看起来比较诡异的功能在匹配 `HTML` 或者 `XML` 标签的时候比较有用比如：

```cpp
>>> re.match(r'<(\w+)>\w+</\1>', '<a>python</a>') is not None
True
>>> re.match(r'<(\w+)>\w+</\1>', '<p>python</p>') is not None
True
>>>
```

# `(?P<name>)` 和 `(?P=name)` 的含义

通常来说在正则表达式中`(...)`代表分组，但是在 python 中`(?...)`表示一种扩展，
比较好用的是`(?P<name>)`和`(?P=name)`这两个。前面提到可以用()分组，然后用
\number 引用组中的内容，但是当分组很多的时候 `\number` 就比较难以维护。Python
中对于这种问题的解法通常都是用名字代替数字比如字符串格式化中的做法

```cpp
>>> '{0} and {1}'.format('Tom', 'Jerry')
'Tom and Jerry'
>>> '{tom} and {jerry}'.format(tom='Tom', jerry='Jerry')
'Tom and Jerry'
>>>
```

正则表达是中也可以用命名的方式替换掉 `\number` 记号，方法就是通过 `(?P<name>)`
给分组起名字然后通过 `(?P=name)` 引用分组。例如：

```cpp
>>> re.match(r'<(?P<TAG>\w+)>\w+</(?P=TAG)>', '<a>python</a>') is not
True
>>>
```

# `search()` 和 `match()` 的区别

`search()` 和 `match()` 的主要区别是 `match()` 只会匹配字符的开头，而
`search()` 会搜索字符的任何位置。另一个区别就是对于 MULTILINE 模式，`^` 标记，
使用 `search()` 可以搜索逻辑行开始而 `match()` 只会匹配物理行开始[^2]。

[^2]: 参考文章第一部分内容

看下面的例子[^3]：

```cpp
# example code:
string_with_newlines = """something
someotherthing"""

import re

print re.match('some', string_with_newlines) # matches
print re.match('someother',
               string_with_newlines) # won't match
print re.match('^someother', string_with_newlines,
               re.MULTILINE) # also won't match
print re.search('someother',
                string_with_newlines) # finds something
print re.search('^someother', string_with_newlines,
                re.MULTILINE) # also finds something

m = re.compile('thing$', re.MULTILINE)

print m.match(string_with_newlines) # no match
print m.match(string_with_newlines, pos=4) # matches
print m.search(string_with_newlines,
               re.MULTILINE) # also matches
```


[^3]: 例子来自 [stackoverflow](http://stackoverflow.com/questions/180986/what-is-the-difference-between-pythons-re-search-and-re-match)

