---
author: 郭荣飞
date: 2015-04-25
title: 解决 jekyll 中文换行变成空格的问题
tags:
 - markdown
 - github pages
 - jekyll
---

# 2015-4-27 更新#

查看 `Liquid` [文档][liquid_doc]发现它有一个默认过滤器 `join`，这和本文编写的过滤器插件名字有冲突
所以把插件中的 `join` 更名为 `join_chinese`。误导之处，海涵。

  [liquid_doc]: https://github.com/Shopify/liquid/wiki/Liquid-for-Designers

# 问题#

使用 `github pages` 搭建一个免费的个人博客是一项非常流行的技术，它有着安全，
免费，无流量限制等特点。

`github pages` 中的文章一般是使用 `Markdown` 编写的，`github pages` 会使用
`jekyll` 把 `Markdown` 编写的文章转换成 `html` 文档从而形成一个静态的博客网站。

在我正儿八经的用 `markdown` 写完一篇文章之前一切都很完美，但是当我开始打开
浏览器查看自己写的文章的时候，却发现了一个非常蛋疼的问题——文章的段落中间总是会
不时出现多余的空格，如下图：

<!--more-->

  ![段落中间的空白][whitespaces]

  [whitespaces]: /img/posts/whitespaces.png

这个问题的出现其实并不是 `github pages` 的引起的，它是由 `Markdown` 和 `HTML`
共同造成的。


# markdown 保留段落中的换行符 #

在把 `.md` 文档装换成 `.html` 文档的过程中，`Markdown` 会保留段落中间的换行符。
为了说明这一点，我们建立下面这个测试文档 `test.md`

	这是同一个段落的第一行
	这是同一个段落的第二行

使用`markdown`命令把它转换成`html`文档

	markdown test.md > test.html

最终得到的 `test.html` 的内容如下：

	<p>这是同一个段落的第一行
	这是同一个段落的第二行</p>

如果你用 `cat -E test.html` 命令查看该文档，会得到以下结果，其中的 `$` 符号
表示换行。

	<p>这是同一个段落的第一段$
	这是同一个段落的第二段</p>$

# HTML 会把换行符转换成空格

浏览器在显示最终的 `.html` 文档的时候会把换行显示成空格，这是一个历史遗留问题。
因为 `HTML` 的语言规范中就是这么规定的。

> An HTML user agent should treat end of line in any of its variations
> as a word space in all contexts except preformatted text

之所以这么规定是因为对于英文来说这是非常重要的一点。比如：

	<p> This is a same
	paragraph</p>

最终的显示效果是

	This is a same paragraph

在英文单词中间加入空格是必须的，因为单词需要通过空格来分界。然而在中文中间加入
空格却是一件非常别扭的事情。他将会导致

	<p> 这是同一个
	段落</p>

被显示成

	这是同一个 段落

这也就是你在 `github pages` 中用 `Markdown` 写的文章最终显示在浏览器上的时候
段落中间会时不时的出现空格的原因——`Markdown`保留换行符而 `HTML` 把换行转换成
空白符。

# 如何解决这个问题

从上面的分析来看，解决这个问题可以四种思路：

 - 在编写 `.md` 文档的时候不要换行，也就是说一个段落只写一行。这样一来就不会有
   换行符的存在，问题也就不会出现。当然这种方式太过于笨拙，相信不会有人想要
   使用它，因为 `Markdown` 设计的理念就是让你的文档易读，易写，这种方式和
   `Markdown` 理念背道而驰。

   这种方式的另外一个缺陷就是如果你使用的编辑器本身会在超过一定长度的时候自动
   换行的话，而你又没有办法更改设置的话，你很难做到不换行。

 - 修改 `Markdown` 实现，让它在生成 `HTML` 的时候去掉段落中间的换行。但是这个
   难度太大，而且如果有 `Markdown` 更新的话，你就得重新修改，很不方便。

 - 第三种解决方案是在浏览器中进行处理。也就是让浏览器不要把换行转换成空白符，
   可选的方式是在 `<html>` 标签中加入 `lang` 属性 `<html lang=zh>`。另外一种
   方式是通过设置 `CSS3` 的 `text-spacing` 属性，`none` 表示不用转换。

   但是这两种方式都不一定有效。因为可能有浏览器不支持,而且 `lang` 属性的设置
   可能会影响默认字体的选择，最终甚至会增加空白字符，因为如果默认字体中的空白
   字符可能很宽[^1]。

   我试过给 `<html>` 添加 `lang` 属性，但是并没有解决问题。

 - 最后一个思路是在 `Markdown` 转换过后的 `HTML` 文件上做文章，把多余的换行符
   清除掉。这也是这篇文章中主要想要介绍的方式。

# 解决方案

`github pages` 使用 `jekyll` 来生成 `html` 文档，所以最好的方式是直接从
`jekyll` 入手解决问题。[解决 Markdown 转 HTML 中文换行变空格的问题][refblog]
一文中提到可以直接修改，`jekyll` 的 `markdown` 转换器，这种方式连作者自己都
不推荐。该文中还提到了另一种方式就是使用 `jekyll` 的 `plugin` 机制，这也是
这篇文章中要介绍的方式。可惜的是这篇文章中提到的 `post_filter.rb` 插件目前已经
不存在了，所以文章介绍的方式无法使用。我们只能自己重新编写插件。

`jekyll` 的插件分为四种，在[官方文档][jekyll]中有详细的介绍。插件是使用 `ruby`
语言编写的，我原来没有 `ruby` 编程经验，所以这里写的插件可能不太规范。关于如何
编写插件可以参考[Getting Started with Jekyll Plugins][plugin_how_to]一文。

我们需要使用到的插件类型是 `Liquid filters`，它的基本框架如下：

```ruby
module Jekyll
    module JoinChineseFilter
        def join_chinese(htmltxt)
        end
    end
end

Liquid::Template.register_filter(Jekyll::JoinChineseFilter)
```

这样我们可以通过 `content` 获得 `html` 文本后使用 `pipeline` 传递给我们编写的
`join_chinese` 进行处理（去掉多余的换行）,
{% raw %} `{{content | join_chinese}}` {% endraw %}

`join_chinese` 最简单的实现方式可能是把所有的 `\n` 都去掉（如果只是想要简单的
去掉所有的行的话，根本不用使用插件，`Liquid` 自带的 `strip_newlines` 过滤器
就可以实现这一点了[^2]）

```ruby
def join_chinese(htmltxt)
    htmltxt.gsub(/\n/, '')
end
```

这样的一个问题就是我们在 `<pre></pre>` 标签中写的 `\n` 也会变替换掉。最直接的
结果就是你引用的代码将会变得一团糟。

其实我们想要去掉的是文本中的换行符，而不是所有的换行符。但是如何才能得到文本
内容并且去掉换行符呢？我最初想要用正则表达式，但是功力不够没能够得到我想要的
结果。最后找到一个非常有用的东西 `nokogiri`。

`nokogiri` 可以把 `html` 转换成一个结构对象，相当于是 `javascript` 中的 `DOM`
对象。有了这个对象之后问题就容易解决的多了。`nokogiri` 的 API 比较的简单。
它的官方网站上提供了一个简单的[教程][noko_tutorial]。

最终插件代码如下：

```ruby
require 'nokogiri'
require 'open-uri'

module Jekyll
	module JoinChineseFilter
		def join_chinese(htmltxt)
			# 生成结构对象
			html_doc = Nokogiri::HTML(htmltxt)

			# 去掉多余的换行
			remove_newline(html_doc.xpath("//body"))
			html_doc.to_html
		end

		private
		def remove_newline(root)
			root.children.each do |child|
				# 跳过 pre 和 code
				next if child.name == 'pre'
				next if child.name == 'code'

				# 如果不是 text 文本节点，递归
				if !child.text?
					remove_newline(child)
				else
					# 如果不是空白文本节点，去掉换行符
					next if child.blank?
					child.content = child.text.gsub(/\n/, '')
				end
			end
		end
	end
end
```
# 注册插件

```ruby
Liquid::Template.register_filter(Jekyll::JoinChineseFilter)
```

把上面这段代码保存为 `joinchinese.rb` 放在 `_plugins` 目录下面，在
`jekyll serve` 启动的时候会自动加载这个插件。然后在我们的 `_layout` 中的各个
`layout` 中使用 {% raw %} `{{content | join_chinese}}` {% endraw %}
最终便可以得到去掉了冗余的空白符的 `html` 文件。效果图如下：

  ![去掉空白][no_whitespace]

# 最后一个难题

问题总算是解决了，很遗憾的是 `github pages` 中的所有页面都是通过 `--safe` 选项
生成，所以你没有办法使用插件，也就是说上面的插件没有办法在 `github pages` 中
使用。

那是不是意味着我们这些努力都白费了呢？当然不是，问题的答案总是伴随着问题一起
诞生的。解决这个问题的方式是你在本地生成页面，然后把生成好的页面 `push` 到
`github` 上，同时使用 `.nojekyll` 文件让 `github page` 不再调用 `jekyll`
生成页面。

这种方法主要有两种实现方式，第一种是使用两个不同的 `repo` 一个放你的源文件
另一个放你生成的页面。第二种是通过一个 `gh-pages` 的分支来完成这一工作。
第一种方式相对来说简单一些，容易上手，而第二种方式更加优雅一些，不过需要有
一定的 `git` 基础知识。你可以参考下面这些文章

 - [使用两个 `repo`][two_ropo]

 - [使用 `gh-pages` 分支][two_branch]

* * * *

  [no_whitespace]: /img/posts/no_whitespaces.png

  [^1]: [Prevent browser converting '\n' between lines into space (for Chinese characters)](http://stackoverflow.com/questions/8550112/prevent-browser-converting-n-between-lines-into-space-for-chinese-characters/8551033#8551033)

  [^2]: [Liquid for Designers](https://github.com/Shopify/liquid/wiki/Liquid-for-Designers)
  [refblog]: http://chenyufei.info/blog/2011-12-23/fix-chinese-newline-becomes-space-in-browser-problem/
  [jekyll]: http://jekyllrb.com/docs/plugins/
  [plugin_how_to]: http://tech.pro/tutorial/1299/getting-started-with-jekyll-plugins
  [noko_tutorial]: www.nokogiri.org
  [two_branch]: http://ixti.net/software/2013/01/28/using-jekyll-plugins-on-github-pages.html
  [two_ropo]: http://charliepark.org/jekyll-with-plugins/
