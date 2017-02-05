---
title: 博客更新小记
date: 2016-01-01 09:20:48
tags:
 - blog
 - hexo
 - next
 - github pages
---

年前在公司实习，和同事说起博客的事情，他推荐我使用 `hexo`。回来一直忙着其他事情
，也就没有尝试去了解 `hexo` 到底是什么玩意。新年忙里偷闲，折腾了一下这技术，可以
说是惊艳，爱不释手。于是把自己在 `github pages` 上面的博客使用 `hexo` 更新了一下
，这里写一个小札记。

<!--more-->

# hexo

个人认为无论是什么样一种技术，了解它的第一步应该是去它的官网看看，那里会有最权威
的信息和文档，`hexo` 也不例外，大家如果有兴趣了解 `hexo`，可以去它的[官网][hexo]
看看。

我不打算在这里介绍 `hexo`，因为我觉得官网上的信息已经足够你了解关于它的任何信息
了。如果你偏好教程类的文章而不太喜欢读文档的话，你可以查看下面这些文章（我粗略的
看过，感觉不算坑）。

[hexo系列教程：](http://www.zipperary.com/2013/05/28/hexo-guide-1/)

# next 主题

我比较喜欢简约的风格，一直以来都认同 `KISS` 原则，所以我选的主题也是一个非常简约
风的 `next`。你可以在 `hexo` 的[官网][hexo]找到各种主题。

`next` 这个主题的有非常高的可配置性，你可以查看它的[使用教程][next]，定制自己的
主题风格。个人觉得需要注意的地方有下面这些：

1. `hexo` 和 `next` 在文档中都把配置写在 `_config` 文件中，但是他们分别有自己的
配置文件。`hexo` 的配置文件在根目录下面，而 `next` 的配置文件在主题的根目录下
面（通常是 themes/next）

2. `next` 有很多 custom.styl 文件可以用来定制你的主题。但是你更改这些文件之后需
要先把这些文件 commit 之后才会在 `github pages` 上起作用。

# hexo 如何托管

`hexo` 可以把生成好的网页一键发布到 `github pages` 上面，这个远比使用 `jekyll`
写文章要方便的多，但是我们如何托管 `hexo` 的源码呢？

通常我们会创建 username.github.io 的仓库，并用它的 master 分支来用来托管 hexo 生
成的网页。你完全可以直接使用另外一个新的仓库比如 blogsource 来托管 hexo 的源码，
不过这样会生成两个不同的仓库，非常不方便。所以我个人趋向于让生成的网页和 hexo 使
用同一个仓库。

个人的做法是在 username.github.io 上面建立一个新的分支 source 用来存放 hexo 源码
，这样你可以在另外一台电脑上把你仓库 clone 下来之后直接切换到 source 分支开始写
博客。

对于 hexo 源码的托管需要注意的地方就是因为 hexo 和 next 属于两个不同的仓库，所以
当我们把 hexo 的源码传到 source 分支后，它不会包含 next 中的文件，只会包含一个空
的文件夹。

你可以在每次克隆完 username.github.io 之后切换到 source 分支下面的 themes 目录，
然后再克隆一份新 next 的代码下来。当然这种做法的前提是你没有自己定制过 next 主题
，因为官方的仓库中必然不会有你的定制代码更新。

我解决这个问题的方法是在自己的账户中 fork 一份 next 官方库，然后把自己的定制修改
提交到自己的仓库当中。以后需要重新 clone 主题的时候直接从自己的 next 仓库中克隆
即可。这样你可以保存自己的代码，同时也可以在 next 更新之后更新自己的私有库。

# hexo 常用插件

使用了一段时间之后碰到了一些问题，不过这些问题通常都可以通过插件解决，下面是我用
的一些插件

## 中文换行变成空格问题

我原来使用 jekyll 的时候就发现了这个问题，最终通过自己编写插件解决了这个问题，并
写了一篇文章阐述问题产生的原因和解决方式，有兴趣的可以看看[解决 jekyll 中文换行
变成空格的问题](/2015/04/25/how-to-fix-the-markdown-newline-blank-problem/)一文
。

hexo 中解决这个问题非常简单，你只需要安装插件
[hexo-filter-fix-cjk-spacing][cjkspacing]即可。

## 脚注问题

原来使用 jekyll 可以使用`[^1]`这样的方式生成脚注，但是在 hexo 中无法实现这一点。
为了能够使用脚注，我使用了[hexo-renderer-markdown-it][markdownit] 这个插件，它的
功能非常强大，推荐大家使用。

不过需要注意的是默认的在 markdown-it 中你没有办法通过 more 来完成摘要的截取，你
需要在配置文件中使用如下的配置：

```cpp
markdown:
render:
html: true
```

具体的可以参考这个链接 https://github.com/hexojs/hexo/issues/1467 和这个链接
https://github.com/celsomiranda/hexo-renderer-markdown-it/issues/14

## 文章目录

安装 [hexo-toc][toc] 插件可以生成文章目录。关于 TOC 这一点，Hexo 的新版本似乎已
经内置支持了，但是我使用 hexo-next 主题的时候会生成目录，但是不会自动给目录选项
加上跳转链接。安装了[hexo-toc][toc] 之后就没有了这个问题。

## 站点部署

旧版本的 Hexo 内置支持 deploy 到 git 仓库中，新版本的需要安装
[hexo-deployer-git][gitdeployer]。

## 搜索功能

文章很多的时候，搜索功能显得非常的重要，hexo 3.0 之后可以使用下面
[hexo-generator-search][search] 建立本地搜索。

[cjkspacing]: https://github.com/lotabout/hexo-filter-fix-cjk-spacing
[markdownit]: https://github.com/celsomiranda/hexo-renderer-markdown-it
[hexo]: https://hexo.io/
[next]: http://theme-next.iissnan.com/
[toc]: https://github.com/bubkoo/hexo-toc
[gitdeployer]: https://github.com/hexojs/hexo-deployer-git
[search]: https://github.com/PaicHyperionDev/hexo-generator-search
