---
layout: post
title:  "异步非阻塞IO"
date:   2015-08-27 23:24:25
categories: jekyll update
---
最近在看nginx。nginx以其卓越的性能在发布后广受欢迎，而它性能之所以高，是因为采用了
事件驱动模型来达到异步非阻塞IO，简单说来就是发送方每个请求在发出后立即返回（异步），接收方在处理这个请求时（进行IO操作），也立即返回（非阻塞），等到IO操作完成，接收方发消息给发送方，并把内容发送给发送方。使用非阻塞的网络IO，特别适合于需要保持长连接的场景，如[long polling],[WebSockets]。

阻塞：当一个函数等待一些事情发生时，会阻塞掉，原因包括网络IO，磁盘IO，mutexes，此时进程挂起。

一些资料：
http://www.zhihu.com/question/19732473
http://blog.csdn.net/historyasamirror/article/details/5778378

[long polling]: https://en.wikipedia.org/wiki/Push_technology#Long_polling
[WebSockets]: https://en.wikipedia.org/wiki/WebSocket
