---
layout: post
title:  "Ningx 中的时间管理"
date:   2015-09-18 14:55:16
categories: jekyll update
---
{% highlight c %}
{% endhighlight %}
Nginx重视程序的高效运行，对耗费时间的系统调用会采取一定的策略。Linux上获取时间的系统调用有`time()`，`gettimeofday()`和`localtime()`函数。Nginx采用的是`gettimeofday`函数。我亲自测了一下，10w次调用大概5ms时间。这个时间开销看起来不大，尤其平均到每一次调用上，大概是50ns，这个调用是如何对服务器性能产生重大影响的呢？

Nginx对时间的策略是采用缓存，只在特定的情况下才会调用`gettimeofday`更新缓存。Nginx的时间缓存会赋值给以下四个变量（均为指针）：
{% highlight c %}
ngx_cached_http_time
ngx_cached_err_log_time
ngx_cached_http_log_time
ngx_cached_http_log_iso8601
{% endhighlight %}

##更新时机
更新时间缓存的两个函数是`ngx_time_update`和`ngx_time_sigsafe_update`。其中以`ngx_time_update`为主。那么何时调用`ngx_time_update`呢？主要有三个：  

1.在Nginx服务器主进程捕捉、处理完一个信号返回的时候；  
2.在缓存索引管理进程中调用该函数，用于标记缓存数据的时间属性；  
3.在工作进程进行事件处理时调用了该函数。  

其中以第三中调用频率最高。可以看下代码：
{% highlight c %}
ngx_epoll_process_events(ngx_cycle_t *cycle, ngx_msec_t timer, ngx_uint_t flags)
{
    //...
    events = epoll_wait(ep, event_list, (int) nevents, timer);

    err = (events == -1) ? ngx_errno : 0;

    if (flags & NGX_UPDATE_TIME || ngx_event_timer_alarm) {
        ngx_time_update();
    }
    //...
}
{% endhighlight %}
可以看到，当flags中包含NGX_UPDATE_TIME标记的时候，或者计时器到时（ngx_event_timer_alarm为1），就会更新缓存。此处有必要说明一下Nginx中的时间分辨率计时器。Nginx允许用户设置计时器的更新频率，也就是每隔多长时间更新一次。如果用户设置了这个值，那么将采用用户的设置来定时更新时间，进行事件处理时不会更新。如果用户没有设置，那么就是在进行事件处理时更新。

##解决读写冲突
以上内容看到了Nginx何时更新缓存。此外，还有一个问题，由于读写缓存时没有加锁，同一个进程对缓存的读写是可能产生冲突的。想象一下这样的场景：一个进程正在执行读缓存操作，读取一半时，收到一个信号，从而中断去执行信号处理函数，而信号处理函数中更新了缓存。当信号处理函数执行完毕返回时后，进程继续读后一半缓存，这样读出来的时间势必是无效的。

Nginx的解决办法是，维护一个时间slot数组，一共64个元素，每次更新缓存都更新数组中一个新的元素，中断之前操作的那个元素并不改变。这样，执行完信号处理函数返回后继续进行读缓存操作时，读到的仍然是之前的slot，这样就不会出错了。此处要注意，上述4个时间缓存都是指针，指针会指向slot中的某一个元素。尽管更新时间后指针的值发生了改变（指向下一个槽），下一个槽的内容（即保存更新后时间的空间）也发生了改变，但进程仍然读到的是之前slot的内存空间，那一部分是没有改变的。
{% highlight c %}
ngx_time_update(void)
{
    u_char          *p0, *p1, *p2, *p3, *p4;
    ngx_tm_t         tm, gmt;
    time_t           sec;
    ngx_uint_t       msec;
    ngx_time_t      *tp;
    struct timeval   tv;

    if (!ngx_trylock(&ngx_time_lock)) {
        return;
    }

    ngx_gettimeofday(&tv);

    sec = tv.tv_sec;
    msec = tv.tv_usec / 1000;

    ngx_current_msec = (ngx_msec_t) sec * 1000 + msec;

    tp = &cached_time[slot];

    if (tp->sec == sec) {
        tp->msec = msec;
        ngx_unlock(&ngx_time_lock);
        return;
    }

    if (slot == NGX_TIME_SLOTS - 1) {
        slot = 0;
    } else {
        slot++;
    }

    tp = &cached_time[slot];

    tp->sec = sec;
    tp->msec = msec;

    ngx_gmtime(sec, &gmt);
    p0 = &cached_http_time[slot][0];
    p1 = &cached_err_log_time[slot][0];
    p2 = &cached_http_log_time[slot][0];
    p3 = &cached_http_log_iso8601[slot][0];
    p4 = &cached_syslog_time[slot][0];
    
    ngx_cached_time = tp;
    ngx_cached_http_time.data = p0;
    ngx_cached_err_log_time.data = p1;
    ngx_cached_http_log_time.data = p2;
    ngx_cached_http_log_iso8601.data = p3;
    ngx_cached_syslog_time.data = p4;

    ngx_unlock(&ngx_time_lock);
}
{% endhighlight %}
在更新缓存时，是需要加锁的。看ngx_trylock的实现可以看到，这个函数类似于操作系统中的test_and_set原子操作。
