---
layout: post
title:  "github page 不再支持jekyll的2.1.4版本"
date:   2016-05-08 19:01:53 +0800
categories: jekyll update
---
今天写博客的过程中，写完一直不能在git上展示。反复尝试改名称，头，都不奏效。只好诉诸暴力解决-升级jekyll。

github pages 是支持jekyll的，也就是说在page下面只要按照jekyll的目录结构设置好就行，生成的目标文件（_site下）也是不需要的。到jekyll的官网可以看到已经升级到3+了，而我本地用的还是2*的版本，猜想是git官方也升级了jekyll，而新的jekyll对旧版本jekyll的支持不够好（居然不能向后兼容，也是醉了）。只好强行升级jekyll。

注意，国内大家一般都用的是淘宝的gem源，淘宝最近把服务升级成了HTTPS，因此，需要替换一下源:
`gem sources --add https://ruby.taobao.org/ --remove http://ruby.taobao.org/`

之后安装更新器：
`gem install rubygems-update`

然后在更新jekyll：
`sudo gem update --system`

于是升级到3.1.3。更新完之后，new一个博客可以看到layout.html文件已经发生了改变。因此很有可能是git的新版jekyll解析器在解析低版本的配置文件时出错了。

