---
layout: post
title:  "霸王餐"
date:   2016-05-08 16:14:37
categories: jekyll update
---
点评的霸王餐是个好东西，一个一个点又比较麻烦，于是需要写一个一键报名所有霸王餐的工具，还有休闲娱乐的，比如健身类的。

以前用Python写过登陆人人网爬好友相册图片的脚本，这个也类似：模拟浏览器操作行为，去选择霸王餐并点击报名。

但第一步就比较麻烦：登陆。一共找到了两个可以登陆的HTML，但是发现点评的登陆，只要是第一次到达页面，输完用户名密码后必然跳出验证码，而找一个识别验证码的程序再调试又比较麻烦，于是作罢。只好求助于cookie。在登陆时，服务器会设置若干个cookie，但经过尝试，发现只需要将dper这个cookie在发送请求时带上即可。

可以登陆之后就比较简单了，到达列表页，抓取所有活动的链接，再遍历这些链接所指向的页面，查找要点击的HTML元素标识，然后模拟点击即可。

写完脚本之后，考虑打包成二进制文件，以便别人在没有Python环境时使用，搜了一下，找到了pyinstaller这个工具：

{% highlight bash%}
#install
sudo pip install pyinstaller
#to binary
pyinstaller -F source.py
#run your binary
./dist/source
{% endhighlight bash%}
