---
layout: post
title:  "Nginx 进程间通信"
date:   2015-09-17 15:32:13
categories: jekyll update
---
知道了socketpair，sendmsg和recvmsg的基本用法，就可以根据源码来梳理一下Nginx中进程是如何通信的。

fork是UNIX创建进程的唯一方法，Nginx当然也不例外。在主进程初始化之后，就会开始创建若干个工作进程，用来处理网络请求事件。主进程有一个保存工作进程信息的数组`ngx_processes`，每一维又有一个`channel`数组，这是一个大小为2的`int`型数组。这个数组就是用来保存socketpair所创建的两个sockfd的。假如进程i要向进程j发消息，那么进程i向`ngx_processes[j].channel[0]`写入消息内容，进程j即可从`ngx_processes[j].channel[1]`读到消息。每个ngx_processes的`channel`都是一对socketpair。  
我们暂且称`ngx_processes[i]`为工作进程i对应的进程槽。

##通信通道的建立
通信通道的建立是与工作进程的创建紧密连在一起的，来看一下相关代码：
{% highlight c %}
static void
ngx_start_worker_processes(ngx_cycle_t *cycle, ngx_int_t n, ngx_int_t type)
{
    ngx_int_t      i;
    ngx_channel_t  ch;

    ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "start worker processes");

    ngx_memzero(&ch, sizeof(ngx_channel_t));

    ch.command = NGX_CMD_OPEN_CHANNEL;

    for (i = 0; i < n; i++) {

        ngx_spawn_process(cycle, ngx_worker_process_cycle,
                          (void *) (intptr_t) i, "worker process", type);

        ch.pid = ngx_processes[ngx_process_slot].pid;
        ch.slot = ngx_process_slot;
        ch.fd = ngx_processes[ngx_process_slot].channel[0];

        ngx_pass_open_channel(cycle, &ch);
    }
}

////
ngx_pid_t
ngx_spawn_process(ngx_cycle_t *cycle, ngx_spawn_proc_pt proc, void *data,
    char *name, ngx_int_t respawn)
{
    if (socketpair(AF_UNIX, SOCK_STREAM, 0, ngx_processes[s].channel) == -1)
    //...
    pid = fork();

    switch (pid) {

    case -1:
    //...
    case 0:
        ngx_pid = ngx_getpid();
        proc(cycle, data);
        break;

    default:
        break;
    }

{% endhighlight %}
在主进程创建工作进程i时，首先用socketpair在进程槽i上创建一对socket，这一对socket就属于进程i，其他进程向i发消息，均是通过该socket的`channel[0]`。之后fork出工作进程，工作进程就开始自己的循环了，而主进程则继续执行，调用`ngx_pass_open_channel`将新进程的socket对传给之前创建的进程。在这里，后面创建的进程已经继承了主进程的进程表，进程表中已经保存了前面已创建进程的接收消息的socket描述符，因此，只需告知之前进程，新创建进程的接收消息socket描述符，这一工作就由`ngx_pass_open_channel`来完成。

让我们来看下`ngx_pass_open_channel`做了什么：  
{% highlight c %}
static void
ngx_pass_open_channel(ngx_cycle_t *cycle, ngx_channel_t *ch)
{
    ngx_int_t  i;

    for (i = 0; i < ngx_last_process; i++) {

        if (i == ngx_process_slot
            || ngx_processes[i].pid == -1
            || ngx_processes[i].channel[0] == -1)
        {
            continue;
        }

        ngx_write_channel(ngx_processes[i].channel[0],
                          ch, sizeof(ngx_channel_t), cycle->log);
    }
}
////////////
typedef struct {
     ngx_uint_t  command;
     ngx_pid_t   pid;
     ngx_int_t   slot;
     ngx_fd_t    fd;
} ngx_channel_t;
{% endhighlight %}
遍历所有工作进程，向他们写一个消息，消息内容封装在类型为ngx_channel_t的ch变量中ch的command域的值为NGX_CMD_OPEN_CHANNEL。其他工作进程使用ngx_read_channel来接收消息，代码如下：
{% highlight c %}
static void
ngx_channel_handler(ngx_event_t *ev)
{
    ngx_int_t          n;
    ngx_channel_t      ch;
    ngx_connection_t  *c;

    if (ev->timedout) {
        ev->timedout = 0;
        return;
    }

    c = ev->data;
    //...
    for ( ;; ) {

        n = ngx_read_channel(c->fd, &ch, sizeof(ngx_channel_t), ev->log);
    switch (ch.command) {
    //...
    case NGX_CMD_OPEN_CHANNEL:
            ngx_processes[ch.slot].pid = ch.pid;
            ngx_processes[ch.slot].channel[0] = ch.fd;
            break;
    //...
    }
{% endhighlight %}
可以看到，接收进程在自己的进程表中设置了发送进程的socket描述符。

{% highlight c %}
{% endhighlight %}


##发送消息：ngx_write_channel
{% highlight c %}
ngx_int_t
ngx_write_channel(ngx_socket_t s, ngx_channel_t *ch, size_t size,
    ngx_log_t *log)
{
    ssize_t             n;
    ngx_err_t           err;
    struct iovec        iov[1];
    struct msghdr       msg;
    //...
    msg.msg_accrights = (caddr_t) &ch->fd;
    //...
    iov[0].iov_base = (char *) ch;
    iov[0].iov_len = size;

    msg.msg_name = NULL;
    msg.msg_namelen = 0;
    msg.msg_iov = iov;
    msg.msg_iovlen = 1;

    n = sendmsg(s, &msg, 0);
}
{% endhighlight %}
这个接口是对sendmsg的封装。Nginx采用ngx_channel_t来存放消息内容，参数以指针传递，ch的指针被赋给了msghdr结构体的iov的iov_base指针。msghdr结构体是sendmsg函数发送消息内容的结构体。

##接收消息：ngx_read_channel
{% highlight c %}
ngx_int_t
ngx_read_channel(ngx_socket_t s, ngx_channel_t *ch, size_t size,
    ngx_log_t *log)
{
    ssize_t             n;
    ngx_err_t           err;
    struct iovec        iov[1];
    struct msghdr       msg;
    //...
    msg.msg_accrights = (caddr_t) &ch->fd;
    //...
    iov[0].iov_base = (char *) ch;
    iov[0].iov_len = size;

    msg.msg_name = NULL;
    msg.msg_namelen = 0;
    msg.msg_iov = iov;
    msg.msg_iovlen = 1;

    n = recvmsg(s, &msg, 0);
    //...
    if (ch->command == NGX_CMD_OPEN_CHANNEL) {
        //...
        ch->fd = fd;
    }
}
{% endhighlight %}
这个接口是对recvmsg的封装。接收到得消息将会保存在msghdr结构体中，ch指针所指向的空间将会被填写。从《UNIX网络编程》来看，这应该是一个深拷贝，即一级指针所指向的内容和二级指针，到最深层次指针所指向的内容都进行了拷贝。ch中的内容都通过socket传递给了接收进程。

如此，就产生了一个疑问，ch.fd已经正确传递，为何还要在另外一个变量（`msg.msg_accrights`）中重新设置一遍，并在读端根据command再把它读出来？岂不是多此一举？
