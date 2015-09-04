---
layout: post
title:  "select,poll and epoll in nginx"
date:   2015-09-04 15:56:39
categories: jekyll update
---
这三种机制本质上均是多路IO在时间上的复用。I/O多路复用就通过一种机制，可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。这种方式比简单的依次在每个描述符上进行阻塞式的等待要高效。但select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写。

##select
select函数原型为：
{% highlight c %}
int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
{% endhighlight %}
第一个参数n表示描述符数，后面三个表示关心的三类事件的描述符列表，最后一个参数表示愿意等待的时间。
当有 I/O 事件到来时， select 通知应用程序有事件到了快去处理，而应用程序必须轮询所有的 FD 集合，测试每个 FD 是否有事件发生，并处理事件。代码如下：
{% highlight c %}
int res = select(maxfd+1, &readfds, NULL, NULL, 120);
if (res > 0)
{
    for (int i = 0; i < MAX_CONNECTION; i++)
    {
	if (FD_ISSET(allConnection[i], &readfds))
	{
	    handleEvent(allConnection[i]);
	}
    }
}
// if(res == 0) handle timeout, res < 0 handle error
{% endhighlight %}

##poll
poll的函数原型为
{% highlight c %}
int poll (struct pollfd *fds, unsigned int nfds, int timeout);

struct pollfd {
int fd; /* file descriptor */
short events; /* requested events to watch */
short revents; /* returned events witnessed */
};
{% endhighlight %}
第一个参数fds表示一个描述符集，第二个参数nfds表示这个集合的大小。每一个pollfd指定描述符编号以及其所关心的状态。revents将记录从内核返回时，发生在fd上的事件。


poll与select的基本工作方式相同，都是先创建一个关注事件的描述符集合，再去等待这些事件发生，然后再轮询描述符集合，检查有没有事件发生，如果有，就进行处理。poll与select的区别在于，select库需要为读事件，写事件，异常事件分别创建一个描述符集合，最后轮训的时候，需要分别轮询这三个集合。而poll库只需要创建一个集合，轮询的时候，可以同时检查这三种事件是否发生。

##epoll
epoll的相关接口如下：
{% highlight c %}
int epoll_create(int size)；
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
typedef union epoll_data {
    void *ptr;
    int fd;
    __uint32_t u32;
    __uint64_t u64;
} epoll_data_t;

struct epoll_event {												                    
    __uint32_t events;      /* Epoll events */														           epoll_data_t data;      /* User data variable */													
    };
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
{% endhighlight %}
`epoll_create`生成一个epoll专用的文件描述符，其实是申请一个内核空间，用来存放你想关注的socket fd上是否发生以及发生了什么事件。size就是你在这个epoll fd上能关注的最大socket fd数。

`epoll_ctl`用来控制某个epoll文件描述符上的事件：注册、修改、删除。其中参数epfd是epoll_create()创建epoll专用的文件描述符。相对于select模型中的FD_SET和FD_CLR宏。

`epoll_wait`用来等待I/O事件的发生。参数说明：  
epfd: 由 epoll_create() 生成的 Epoll 专用的文件描述符；  
epoll_event: 用于回传代处理事件的数组；  
maxevents: 每次能处理的事件数；  
timeout: 等待 I/O 事件发生的超时值；  
返回发生事件数。  
可以看到，epoll_wait返回时，可以将待处理事件直接返回，而不必再去轮询所有描述符，因此当描述符（Nginx中的socket）数量非常大时，这种方式非常高效。用法如下：
{% highlight c %}
int res = epoll_wait(epfd, events, 20, 120);
for (int i = 0; i < res;i++)
{
    handleEvent(events[n]);
}
{% endhighlight %}
epoll的优点之处体现在以下几个方面：  

1. 监视的描述符数量不受限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在1GB内存的机器上大约是10万左 右，具体数目可以cat /proc/sys/fs/file-max察看,一般来说这个数目和系统内存关系很大。select的最大缺点就是进程打开的fd是有数量限制的。这对 于连接数量比较大的服务器来说根本不能满足。虽然也可以选择多进程的解决方案( Apache就是这样实现的)，不过虽然linux上面创建进程的代价比较小，但仍旧是不可忽视的，加上进程间数据同步远比不上线程间同步的高效，所以也 不是一种完美的方案。  

2. IO的效率不会随着监视fd的数量的增长而下降。epoll不同于select和poll轮询的方式，而是通过每个fd定义的回调函数来实现的。只有就绪的fd才会执行回调函数。  

3. mmap加速内核与用户空间的信息传递。epoll是通过内核于用户空间mmap同一块内存，避免了无畏的内存拷贝。  

##其他
这三种方式只能用于网络套接字的情形，不能用于regular file。对regular file的open，read之类的操作都是一定会导致阻塞。要想达到非阻塞，则需要用aio。

regular file:纯文本或二进制文件。其他，如pipe，socket，directory，symbolic link不是regular file。
