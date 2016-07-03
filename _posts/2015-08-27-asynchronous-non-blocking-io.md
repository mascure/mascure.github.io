---
layout: post
title:  "异步非阻塞IO"
date:   2015-08-27 23:24:25
categories: jekyll update
---
最近在看nginx。nginx以其卓越的性能在发布后广受欢迎，而它性能之所以高，是因为采用了
事件驱动模型来达到异步非阻塞IO。由于Nginx本质上是运行在一台Linux机器上的一个进程，它所涉及到的只能是一些运算，和IO（磁盘IO或socket IO）。因此此处的异步非阻塞IO只能涉及到两个系统对象，一个是调用这个IO的进程，另一个是系统内核（只有内核才有权限执行IO）。因此阻塞与否谈得是调用这个IO的进程会不会阻塞，而同步异步谈的是进程和内核之间是否存在等待。  
当一个read操作发生时，会经历两个阶段：  
1 等待数据准备  
2 将数据从内核拷贝到进程中  

根据stackoverflow上的[这个解释]，阻塞和同步可以理解为同一个事物，即进程调用一个API，进程本身被挂起直到内核返回结果。而非阻塞和异步则不同，非阻塞意味着如果结果不能立即返回，那么API立即返回一个错误并且什么也不做，因此需要进程有某种查询的方法来看API是否可以被调用。而异步意味着API总是立即返回，并在“后台”开始准备结果，准备完成后通知进程。  

所以一个阻塞式的IO是这样的：
![blocking IO](/assets/images/blocking_IO.gif)  

一个同步非阻塞式的IO是这样的：
![non-blocking IO](/assets/images/non_blocking_IO.gif)  

非阻塞比阻塞优势在于，进程不必等待数据准备好，并且可以过一段时间查询一次，而这段时间可以用来做别的事，提高了效率。

一个异步非阻塞式的IO是这样的： 
![asynchronous_non-blocking IO](/assets/images/asynchronous_non_blocking_IO.gif)  

可以看到，进程读调用结束后立即返回，内核将会等待数据准备好，然后将数据拷贝到进程空间，再发信号给进程，进程处理函数将会处理结果。这种方式比同步非阻塞的方式优势在于，不必反复查询数据是否准备好，也不用等待数据从内核拷贝到进程。

因此，对于大量的访问请求，第三种方式，即异步非阻塞的IO无疑是最高效的。

一些资料:        

<http://www.zhihu.com/question/19732473>  
<http://blog.csdn.net/historyasamirror/article/details/5778378>

[这个解释]: http://stackoverflow.com/questions/2625493/asynchronous-vs-non-blocking
