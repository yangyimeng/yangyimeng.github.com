---
layout: post
title: "修改jekyll的code block style"
description: ""
category: 'work linux script'
tags: []
sumarry: |
  现在流行用jekyll来运行个人的博客网站, 是一个小型的静态服务器.用的是ruby语言来实现的.笔者的博客也是用jekyll来搭建的,用起来很方便,很有装逼的感觉,呵呵,废话少说.马上进入正题.jekyll博客系统,用来markdown来方便用户进行写作,主要是让用户可以随意的写作,就可以很好的显示在网页上面,当中对于代码的显示,用了code  block来特意的标明代码区, 但是原始的代码块没有高亮显示语法,因此用起来不爽.因此,笔者的文章的目的就在于使得jekyll的默认的code block具有语法高亮.
---
{% include JB/setup %}

  现在流行用jekyll来运行个人的博客网站, 是一个小型的静态服务器.用的是ruby语言来实现的.笔者的博客也是用jekyll来搭建的,用起来很方便,很有装逼的感觉,呵呵,废话少说.马上进入正题.jekyll博客系统,用来markdown来方便用户进行写作,主要是让用户可以随意的写作,就可以很好的显示在网页上面,当中对于代码的显示,用了code  block来特意的标明代码区, 但是原始的代码块没有高亮显示语法,因此用起来不爽.因此,笔者的文章的目的就在于使得jekyll的默认的code block具有语法高亮.

#markdown的认知

 Markdown 的目标是实现「易读易写」, 而jekyll就是使用了Markdown来作为文章的书写语言,我们需要来关注的是markdown的一个特殊的功能.

##markdown代码区块
 和程序相关的写作或是标签语言原始码通常会有已经排版好的代码区块，通常这些区块我们并不希望它以一般段落文件的方式去排版，而是照原来的样子显示，Markdown 会用 &lt;pre&gt;和&lt;code&gt;标签来把代码区块包起来。

要在 Markdown 中建立代码区块很简单，只要简单地缩进 4 个空格或是 1 个制表符就可以，例如，下面的输入：
<pre class="brush: js;">
	这是一个普通段落：
	    这是一个代码区块。
</pre>
Markdown 会转换成：
<pre class="brush: js;">
	<p>这是一个普通段落：</p>
	<pre><code>这是一个代码区块。
	</code></pre>
</pre>
#改造的目标样式

通过上面, 我们知道markdown会识别出缩进 4 个空格或是 1 个制表符的特殊区块,然后用pre标签进行封装,以显示出代码的特效.只不过呢,原始的特效不好看,因此,我们需要对这个特效进行修改.

上面的说明让我们知道,我们有办法对默认的样式进行修改,但是首先要找到一个好看的样式来替换旧的样式.


##SyntaxHighlighter

SyntaxHighlighter是一个开源的html高亮的css项目,效果如下,怎样,挺不错的吧.
<pre class="brush: js;">
	// SyntaxHighlighter makes your code snippets beautiful without tiring your servers.
	// http://alexgorbatchev.com
	var setArray = function(elems) {
	    this.length = 0;
	    push.apply(this, elems);
	    return this;
	}
</pre>




#开始改造markdown

现在我们既知道可以修改旧样式,又有了我们的新样式, 恩,不错,我们已经可以开始动工了.


##下载SyntaxHighLighter,并放入jekyll网站项目里
[下载](http://alexgorbatchev.com/SyntaxHighlighter/)SyntaxHighLighter

笔者使用的是[syntaxhighlighter_3.0.83](http://alexgorbatchev.com/SyntaxHighlighter/download/download.php?sh_current)

下载后,把压缩包里的四个文件夹 compass, scripts, src, styles 全部的都放到jekyll的assets/themes/your-theme-name路径下面.


最后, 要在default.html里面include一些必要的css,js文件.如下:

<pre class="brush: js;">
	  <script type="text/javascript" src="{{ ASSET_PATH }}/scripts/shCore.js"></script>
	  <script type="text/javascript" src="{{ ASSET_PATH }}/scripts/shBrushJScript.js"></script>
	  <link type="text/css" rel="stylesheet" href="{{ ASSET_PATH }}/styles/shCoreDefault.css"/>
	  <script type="text/javascript">SyntaxHighlighter.all();</script>
</pre>


##找到要修改的源码的地方

由于笔者使用的是ubuntu系统,因此可以通过如下的命令来找到要修改的源码的地方.
<pre class="brush: js;">
	xxx@ubuntu:~$ which jekyll 
	/home/xxx/.rvm/gems/ruby-1.9.2-p320/bin/jekyll
</pre>
呵呵,其中xxx就是你的用户名了, 不错, 源码的路径就应该是在/home/xxx/.rvm/gems/ruby-1.9.2-p320这个地方附近了.


一般的话, 要修改的文件的路径应该是
<pre class="brush: js;">
	/home/xxx/.rvm/gems/ruby-1.9.2-p320/gems/maruku-0.6.1/lib/maruku/output/to_html.rb
</pre>

很好, 我们的工作进入到了很关键的时刻了, 因为我们就要开始动手修改ruby markdown的源码了.呵呵,事实上, 我们所要修改的ruby的源码只是很少的一部分.

请把这个文件的下面的函数改造成下面的代码,就ok了.很简单的,看看就懂了.
<pre class="brush: js;">
		def to_html_code_using_pre(source)
			pre = create_html_element  'pre'
			s = source
			pre.attributes['class'] = 'brush: js;'		
			s = Text.normalize(s)
			s  = s.gsub(/\&apos;/,'&#39;') # IE bug
			s  = s.gsub(/'/,'&#39;') # IE bug
			if get_setting(:code_show_spaces) 
				# 187 = raquo
				# 160 = nbsp
				# 172 = not
				s.gsub!(/\t/,'&#187;'+'&#160;'*3)
				s.gsub!(/ /,'&#172;')
			end
			text = Text.new(s, respect_ws=true, parent=nil, raw=true )
			pre &lt;&lt; text
			pre
		end
</pre>

#看看改造的效果吧.

呵呵,效果就在上面, 你们只要对想要显示的代码缩进四个空格或者一个制表,就可以显示出上面的特效了.

不过呢, 有一点需要各位注意的是, 若是你要把博客挂在github上的话, 那你修改的只是本地的markdown,可是github上的markdown你并没有修改到,因此,效果就会出不来.笔者过后会用另外的方法来介绍一下,若是要把博客挂在github上,又要显示出上面的特效的方法.

##参考文章
* [ruby参考文档](http://ruby-doc.org/)
* [ruby教程](http://pine.fm/LearnToProgram/)
