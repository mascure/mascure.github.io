---
layout: post
title:  "为博客添加评论"
date:   2015-08-13 15:01:39
categories: jekyll update
---
去国外绕了一圈，还是回来了。

最开始搜索jekyll comment，搜到的是[这个]，可是在按照步骤配置好以后，页面上的评论部分是显示出来了，但点击评论时会提示不允许使用post方法。这让我很诧异，难道jekyll不允许使用post方法，还是comment部分在接收时出了问题，去网上找解决办法，因为这是mpalmer在git上的一个项目，受众比较小，没有搜到有价值的回答。自己也没心情去研究它的代码。于是只好换别的评论系统。

然后就看到disqus，一个第三方的评论管理平台，这才意识到mpalmer的评论其实是存在本地的，那就会有一个很大的优势：速度快。大概了解了下disqus的安装步骤之后，就开始安装。可是第一步就卡住了，disqus.com页面始终刷不出来，包括使用代理。于是可以想到这样的画面，即使安装成功，在现实博客的时候，评论部分始终在加载，标题上的那个圈圈始终在转，这是多么令人不爽。于是决定，找国内的类似平台。

于是找到了[多说]，用法就是把一段代码插入到博客内容的下面。jekyll用的是模板，所以，只需在模板里面加入这段代码就行了。
代码如下。把头部的三个变量按提示替换掉即可(把`\`去掉，为什么后面会说)。

发现国内的果然加载超级快，哈哈。

_includes/comment.html
{% highlight html %}
<!-- 多说评论框 start -->
	<div class="ds-thread" data-thread-key="\{\{page.id\}\}" data-title="\{\{page.title\}\}" data-url="\{\{site.url\}\}\{\{page.id\}\}"></div>
<!-- 多说评论框 end -->
<!-- 多说公共JS代码 start (一个网页只需插入一次) -->
<script type="text/javascript">
var duoshuoQuery = {short_name:"mascure"};
	(function() {
		var ds = document.createElement('script');
		ds.type = 'text/javascript';ds.async = true;
		ds.src = (document.location.protocol == 'https:' ? 'https:' : 'http:') + '//static.duoshuo.com/embed.js';
		ds.charset = 'UTF-8';
		(document.getElementsByTagName('head')[0] 
		 || document.getElementsByTagName('body')[0]).appendChild(ds);
	})();
	</script>
<!-- 多说公共JS代码 end -->
{% endhighlight %}

在`_layouts/post.html`文件的末尾，加入代码`\{\% include comment.html \%\}`。
（把`\`去掉）

在搭建的过程中，一直有一个疑惑，类似site.url,page.id这样的变量并没有在哪个文件里面设置（用grep查找），却可以在页面里使用，这是为什么。于是猜测这是jekyll定义的变量。jekyll的[官方文档]，可以看到有三类变量，global，site，page，在页面中可以用`\{\{变量名\}\}`的方法使用，在生成静态页面的时候，jekyll会把所有这样的变量替换为对应的值。所以上文里要加`\`以防止被替换掉。

[这个]: https://github.com/mpalmer/jekyll-static-comments/
[多说]: duoshuo.com/
[官方文档]: http://jekyllrb.com/docs/variables/
