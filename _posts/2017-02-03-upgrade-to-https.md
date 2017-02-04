---
layout: post
title:  "升级到HTTPS"
date:   2017-02-03 19:39:24
categories: jekyll update
---
升级到https的过程，参考了这篇文章<https://sheharyar.me/blog/free-ssl-for-github-pages-with-custom-domains/>.

Git Page本身是支持https访问的，如果没有自定义域名，可以直接访问https://yourname.github.io。但我的域名是mascure.info。https的证书是和域名绑定的。因此，如果想让mascure.info可以支持https访问，应当为mascure.info配置一个证书。通过Google，发现了一个更加釜底抽薪的办法，把mascure.info域名的管辖权托管给第三方网站，由第三方网站负责https证书的所有事情。这就是上面文章中提到的办法。

众所周知，所谓https就是浏览器和服务器先建立一个安全连接，之后所有的http请求和相应均通过这个通道加密传输。因此，浏览器需要先和mascure.info的服务器建立安全连接，这个服务器需要拥有mascure.info域名的证书。按照目前的过程，假如浏览器想通过https访问mascure.info，会走到mascure.github.io，但是它并没有mascure.info的证书，因此无法提供https服务。

通过上面的方法，第三方网站为mascure.info配置了证书，并且扮演了mascure.info服务器，与浏览器建立https通道，然后从mascure.github.io获取对应的资源返回给浏览器，这一段只能通过http。

所以，使用上述方法配置后，从浏览器看到的https://mascure.info，发起的请求实质上经历了1、浏览器与第三方网站的https通道，2、第三方网站与mascure.github.io的http通道。因此实际上是伪https（通信内容并不是全程加密）。

##### 让多说支持https #####
配置完之后，首页上的地址栏已经有绿色的锁，但文章页面上有个i，提示有图片非https。排除文章内容本身，打赏部分，只剩下评论部分，发现新浪的头像并不是https的。通过Google，发现其实是用的多说插件这里做的不好，但迟迟没有修复这个问题，不过网上提供了另一种解决办法：<https://github.com/rainwsy/duoshuo-https>。

可惜这次迁移并不是平滑迁移，大概一个小时之后才可以访问。

搞定，于是全站已经https。
