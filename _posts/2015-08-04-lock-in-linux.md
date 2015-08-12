---
layout: post
title:  "lock in linux"
date:   2015-08-04 13:17:58
categories: jekyll update
---
昨天某厂面试，问到Linux中锁的用法，于是回来巩固（学习）了一下。
使用锁除了保证进程（线程）间对变量的互斥访问外，还可以达到进程（线程）同步。

Linux中有自旋锁，信号量，互斥量三种锁，互斥量只能用于线程间，讲解见[这里]，信号量可以用于进程间，也可以用于线程间，[进程间用法]，[线程间用法]。[自旋锁用法]。

[这里]:http://blog.csdn.net/ljianhui/article/details/10875883
[进程间用法]:http://blog.csdn.net/ljianhui/article/details/10243617
[线程间用法]:http://blog.csdn.net/ljianhui/article/details/10813469
[自旋锁用法]:http://www.cnblogs.com/biyeymyhjob/archive/2012/07/21/2602015.html

线程间同步：

互斥量：

{% highlight c++%}
#include <pthread.h>  

int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *mutexattr);

int pthread_mutex_lock(pthread_mutex_t *mutex);

int pthread_mutex_unlock(pthread_mutex_t *mutex);

int pthread_mutex_destroy(pthread_mutex_t *mutex);
{% endhighlight %}
用法：
使用init初始化一个mutex，然后lock与unlock之间，可以获得这个锁，其他想获得这个锁的线程会挂起。结束的时候destroy。下面是一个控制主线程和子线程交替执行的例子。

{% highlight c++%}
pthread_mutex_t mutex;

int main(){
    ...
    res = pthread_mutex_init(&mutex, NULL);
res = pthread_create(&thread, NULL, thread_func, msg);
lock(mutex);
while(condition){
    ...
    unlock();
    sleep(1);
    lock();
}
}

void* thread_func(void *msg){
lock();
while(condition){
    ...
    unlock();
    sleep(1);
    lock();
}
}

{% endhighlight %}

使用信号量控制主线程和子线程交替执行的例子。

{% highlight c++%}
//声明信号量
sem_t sem1,sem2;

int main{
//初始化信号量
res = sem_init(&sem1, 0, 0);
res = sem_init(&sem2, 0, 1);
res = pthread_create(&thread, NULL, thread_func, msg);
sem_wait(&sem2);
while(condition){
...
sem_post(&sem1);
sem_wait(&sem2);
}
}

void* thread_func(void *msg){
sem_wait(&sem1);
while(condition){
...
sem_post(sem2);
sem_wait(sem1);
}
sem_post(sem2);
}
{% endhighlight %}
