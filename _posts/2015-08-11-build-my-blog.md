---
layout: post
title:  "build my blog"
date:   2015-08-11 13:17:58  
categories: jekyll update
---
后续要做：博客front matter的date按F5自动加入；把以前的博客迁移过来；添加评论；

建立了自己的博客，终于有自己的一片天地了。

我是在github pages存放博客内容的，用jekyll生成的静态博客，然后把github.io的域名绑定到新买的域名，也就是现在的mascure.info上。
未完待续，要睡觉了。

#0、以前的博客#
我之前已经借助[isnowfy]在github page上搭建了一个静态博客，挺好用的一个工具，可以在线编辑文章，然后发布，这个工具会自动把你的内容生成静态页面发到你的git上去（需要提供用户名密码）。另外就是用到了markdown语法，很简洁的一个类似于html的标记语言。但毕竟是他自己写的一个工具，其他特性不能支持，比如更改主题，添加评论等。于是切换到jekyll上，也是非常简单易用。

[isnowfy]: http://isnowfy.github.io/about-simple-cn.html

#1、git上创建名为mascure.github.io的project#
github 为每个账号提供了300M的空间，用以用户或组织的介绍，这个空间只能对应于username.github.io的项目，该项目里面只要提供了网站相应内容，用户就可以通过username.github.io的域名访问。这一步在步骤0上，isnowfy的工具已经帮我做了。这一步做完后clone到本地来，比如我的是mascure.github.io的文件夹。此时文件夹里面应该只有一个README.md文件。

#2、安装jekyll#
安装非常简单，
{% highlight bash%}
$ gem install jekyll
{% endhighlight %}
但gem使用的默认的源是https://rubygems.org，因为国内网络原因（你懂的），导致 rubygems.org 间歇性连接失败，所以敲了命令后半天没有反应，可以用`gem install rails -V`来查看执行过程。需要将源切换到国内淘宝的源，具体步骤如下：
{% highlight bash%}
$ gem sources --remove https://rubygems.org/
$ gem sources -a https://ruby.taobao.org/
$ gem sources -l
*** CURRENT SOURCES ***

https://ruby.taobao.org
# 请确保只有 ruby.taobao.org
$ gem install jekyll
{% endhighlight %}

#3、初始化博客目录#
这一步是按照jekyll的目录结构初始化mascure.github.io的文件夹。可以用`jekyll new myblog`先创建一个文件夹，然后把该目录下所有文件拷贝到mascure.github.io。

#4、编辑，预览，发布#
此时可以看到目录中有_posts，_layouts等文件夹，写博客是在_posts下，可以看到有一篇2015-08-08-welcome-to-jekyll.markdown的文章，可以参照这篇文章来写。按照格式要求写好front matter，然后就是用markdown语法写博文了。文件名也必须严格符合要求的格式。此时可以敲入`jekyll serve`就在本地建立了一个web服务器，浏览器用`http://localhost:4000`就可以访问网站了。这个主要是一个预览的功能，能够在本地就看一下效果，并且每一次更改，jekyll都可以检测到，并生成对应的静态页面（在_sites目录下）。如果不想在本地预览直接发布的话，可以使用`jekyll build`生成静态页面即可。

生成好静态博客后，就可以推送到git上面去了，步骤如下
{% highlight bash%}
$ git add . --all
$ git commit -m 'new blog'
$ git push origin master
{% endhighlight %}
git默认使用的协议是http的，可以用`git remote -v`看到。这个会导致push失败，所以要改成ssh协议，步骤如下：
{% highlight bash%}
$ git remote rm origin
$ git add origin git@github.com:mascure/mascure.github.io.git
$ git push origin master
{% endhighlight %}

#关于设置独立域名#
网站设置好以后，用的是mascure.github.io的域名访问。要想设置独立域名的话，比如我用mascure.info访问，只需在根目录下添加一个文件CNAME，里面只写一行，就是mascure.info。然后就是把mascure.info的DNS provider中添加一条CNAME的记录，具体步骤见[这里]。

[这里]:https://help.github.com/articles/setting-up-a-custom-domain-with-github-pages/
