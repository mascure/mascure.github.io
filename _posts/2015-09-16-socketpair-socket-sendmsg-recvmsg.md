---
layout: post
title:  "socketpair,socket,sendmsg和recvmsg"
date:   2015-09-16 10:07:33
categories: jekyll update
---
之前没有接触过socket，最近看Nginx，也顺便弥补一下这方面的不足。

##socketpair
这个用于创建在本地通信的两个进程之间的socket。调用结束将会创建两个文件描述符，向其中一个写数据，就可以从另一个读。通常的做法是，先用socketpair建立一个连接的socket，然后fork出一个子进程，子进程继承了这个socket，主进程从一端读写，子进程在另一端读写。函数如下：
{% highlight c %}
#include <sys/types.h>
#include <sys/socket.h>
/*
The socketpair() call creates an unnamed pair of connected sockets in
the specified domain, of the specified type, and using the optionally
specified protocol.
The descriptors used in referencing the new sockets are returned in
sv[0] and sv[1].  The two sockets are indistinguishable.
On success, zero is returned.  On error, -1 is returned, and errno is
set appropriately.
*/
int socketpair(int domain, int type, int protocol, int sv[2]);
{% endhighlight %}
一个示例的创建及读写程序如下：
{% highlight c %}
#include <sys/types.h>
#include <sys/socket.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>

int main ()
{
    int fd[2];
    
    int r = socketpair( AF_UNIX, SOCK_STREAM, 0, fd );
    if ( r < 0 ) {
        perror( "socketpair()" );
        exit( 1 );
    }
    
    if ( fork() ) {
        /* Parent process: echo client */
        int val = 0;
        close( fd[1] );
        while ( 1 ) {
            sleep( 1 );
            ++val;
            printf( "Sending data: %d\n", val );
            write( fd[0], &val, sizeof(val) );
            read( fd[0], &val, sizeof(val) );
            printf( "Data received: %d\n", val );
        }
    }
    else {
        /* Child process: echo server */
        int val;
        close( fd[0] );
        while ( 1 ) {
            read( fd[1], &val, sizeof(val) );
            ++val;
            write( fd[1], &val, sizeof(val) );
        }
    }
}
{% endhighlight %}
在这个程序中，read是阻塞式的，即当没有数据可读时，会导致进程挂起。这就会让人联想到另一个问题，假如write的缓冲区写满了，会怎样？我去查了一下，发现write对于不同的文件类型，会有不同的处理，具体可以看[write 文档]。在本例中，对应于`If fildes refers to a socket, write() shall be equivalent to send() with no flags set.`。对于send来说，如果打开文件时没有设置O_NONBLOCK，那么当空间不足时，send会阻塞，直到空间足够。因此在本例中，read和write都是阻塞式的。  

另外，这一对文件描述符是可以在同一个进程内读写的，比如父进程从fd[0]写，就可以从fd[1]读到。还有一点是，假如close 0（使得fd[0]的引用计数为0），那么当从fd[1]read时，读到的是EOF。  

文件的O_NONBLOCK标志可以在打开时设置，也可以打开后用fcntl函数改变文件的特性，这些特性除了O_NONBLOCK外，还有读、写、追加等。

[write 文档]: http://linux.die.net/man/3/write

##socket(提到connect,bind,listen,accept）
这个函数创建一个套接字，返回一个小的非负整数值，与文件描述符类似，称为套接字描述符（socket descriptor），简称sockfd。函数如下：
{% highlight c %}
#include <sys/types.h>
#include <sys/socket.h>
/*
socket() creates an endpoint for communication and returns a
descriptor.
On success, a file descriptor for the new socket is returned.  On
error, -1 is returned, and errno is set appropriately.
*/
int socket(int domain, int type, int protocol);
{% endhighlight %}
一个基本的TCP客户/服务器通信的流程如下：
![TCP_client_server](/images/tcp_client_server.png)

根据上图，一个使用socket的典型的客户端服务器程序如下（功能：获取服务器时间）：
{% highlight c %}
//client
#include	"unp.h"
int
main(int argc, char **argv)
{
	int					sockfd, n;
	char				recvline[MAXLINE + 1];
	struct sockaddr_in	servaddr;

	if (argc != 2)
		err_quit("usage: a.out <IPaddress>");

	if ( (sockfd = socket(AF_INET, SOCK_STREAM, 0)) < 0)
		err_sys("socket error");

	bzero(&servaddr, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_port   = htons(13);	/* daytime server */
	if (inet_pton(AF_INET, argv[1], &servaddr.sin_addr) <= 0)
		err_quit("inet_pton error for %s", argv[1]);

	if (connect(sockfd, (SA *) &servaddr, sizeof(servaddr)) < 0)
		err_sys("connect error");

	while ( (n = read(sockfd, recvline, MAXLINE)) > 0) {
		recvline[n] = 0;	/* null terminate */
		if (fputs(recvline, stdout) == EOF)
			err_sys("fputs error");
	}
	if (n < 0)
		err_sys("read error");

	exit(0);
}

//server
#include	"unp.h"
#include	<time.h>
int
main(int argc, char **argv)
{
	int					listenfd, connfd;
	struct sockaddr_in	servaddr;
	char				buff[MAXLINE];
	time_t				ticks;

	listenfd = Socket(AF_INET, SOCK_STREAM, 0);

	bzero(&servaddr, sizeof(servaddr));
	servaddr.sin_family      = AF_INET;
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
	servaddr.sin_port        = htons(13);	/* daytime server */

	Bind(listenfd, (SA *) &servaddr, sizeof(servaddr));

	Listen(listenfd, LISTENQ);

	for ( ; ; ) {
		connfd = Accept(listenfd, (SA *) NULL, NULL);

        ticks = time(NULL);
        snprintf(buff, sizeof(buff), "%.24s\r\n", ctime(&ticks));
        Write(connfd, buff, strlen(buff));

		Close(connfd);
	}
}
{% endhighlight %}
注：程序源自《UNIX网络编程》

##sendmsg,recvmsg
这两个函数是最通用的I/O函数，可以把所有read，readv，recv和recvfrom调用替换成recvmsg调用。输出函数也可以替换成sendmsg调用。相应地几个输出函数之间的对比如下：

函数        增加的特性  
write        最简单的套接口写入函数  
send        增加了flags标记  
sendto        增加了套接口地址与套接口长度参数  
writev        没有标记与套接口地址，但是具有分散写入的能力  
sendmsg        增加标记，套接口地址与长度，分散写入以及附属数据的能力

{% highlight c %}
#include <sys/socket.h>
ssize_t recvmsg(int sockfd, struct msghdr *msg, int flags);
ssize_t sendmsg(int sockfd, struct msghdr *msg, int flags);
//返回：若成功则为读入或写出的字节数，若出错则为-1
{% endhighlight %}

