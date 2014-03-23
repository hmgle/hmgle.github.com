---
layout: post
title:  "Linux 多进程多路复用唤醒冲突"
date:   2014-03-23 20:57:43
categories: Linux select poll epoll thundering herd
---

Linux 对于 `accept(2)` 的**惊群**(thundering herd)问题，
早已解决。目前许多人也把这种现象称为新的惊群：用多路复用模型时,
不同的进程监控的文件描述符集合的交集不为空， 等这个交集的某个文件IO事件触发后，
内核将的多个监控了这个io且阻塞在 `select(2)`, `poll(2)` 或
`epoll_wait(2)` 的进程唤醒。但严格来说， 这种现象不叫惊群(thundering herd),
而是**冲突**(collision). 对于内核来说，
唤醒所有监控这一IO事件的进程是合理的。这是因为: select/poll/epoll 不同与
accept， 它们监控的文件描述符是可以被多个进程同时处理的，
比如一个进程只读取这个文件句柄一小部分数据， 另一进程读剩余部分。而 accept
处理的套接字是互斥的，一个套接字不能被两个进程 accept.

我注意到，对这种 **select/poll/epoll**
冲突的理解存在许多误区，比如有人都用如下类似的代码模拟select冲突(网上搜 select
惊群或 epoll 惊群有真相):

{% highlight c linenos %}
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <stdlib.h>
#include <strings.h>
#include <arpa/inet.h>

void worker_hander(int listenfd)
{
	fd_set rset;
	int connfd, ret;

	printf("worker pid#%d is waiting for connection...\n", getpid());
	for (;;) {
		FD_ZERO(&rset);
		FD_SET(listenfd,&rset);
		ret = select(listenfd+1,&rset,NULL,NULL,NULL);
		if(ret < 0)
			perror("select");
		else if(ret > 0 && FD_ISSET(listenfd, &rset)) {
			printf("worker pid#%d 's listenfd is readable\n",
					getpid());
			connfd = accept(listenfd, NULL, 0);
			if(connfd < 0) {
				perror("accept error");
				continue;
			}
			printf("worker pid#%d create a new connection...\n",
					getpid());
			sleep(1);
			close(connfd);
		}
	}
}

static int fd_set_noblock(int fd)
{
	int flags;

	flags = fcntl(fd, F_GETFL);
	if (flags == -1)
		return -1;
	flags |= O_NONBLOCK;
	flags = fcntl(fd, F_SETFL, flags);
	return flags;
}

int main(int argc,char*argv[])
{
	int listenfd;
	struct sockaddr_in servaddr;
	int sock_opt = 1;

	listenfd = socket(AF_INET,SOCK_STREAM,0);
	if (listenfd < 0) {
		perror("socket");
		exit(1);
	}
	fd_set_noblock(listenfd);
	if ((setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, (void *)&sock_opt,
			sizeof(sock_opt))) < 0) {
		perror("setsockopt");
		exit(1);
	}
	bzero(&servaddr, sizeof servaddr);
	servaddr.sin_family = AF_INET;
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
	servaddr.sin_port = htons(1234);
	bind(listenfd, (struct sockaddr*)&servaddr, sizeof(servaddr));
	listen(listenfd, 10);

	pid_t pid;
	pid = fork();
	if (pid < 0) {
		perror("fork");
		exit(1);
	} else if (pid == 0)
		worker_hander(listenfd);
	worker_hander(listenfd);
	return 0;
}
{% endhighlight %}

编译后用先运行以上的服务端， 客户端可以用 `netcat` 模拟连接:

	nc 127.0.0.1 1234

以上代码是两个进程同时监控同一个文件描述符，返回的结果基本是只有一个select返回。于是试验人认为"并不是将所有工作进程全部唤醒，而只是唤醒了一部分".

这个错误的认识在于没有理解**唤醒**的含义. 并不是要从 `select(2)` 返回才叫唤醒。

一个进程在等待的io事件发生之前，内核会为这个进程描述符的state字段设置
**TASK_INTERRUPTIBLE** 状态,
此时进程描述符位于等待队列中。一旦等待的事件发生后，进程就会被**唤醒**，进程描述符就会被移到运行队列中，
发生进程切换时，内核进程调度器会根据调度策略从运行队列选择一个进程执行。

因此，上述程序实际上**唤醒**了所有的两个进程, 只不过先被调度的那个进程
`select(2)` 返回后， 如果执行到`accept(2)`
也没有发生进程切换，把IO事件处理掉了。而等到后调度的那个进程执行时，`select(2)`
里面已经没有这个IO事件了,
内核检测这个进程没有监控的事件发生，会把这个进程继续放到等待队列里面去,
`select(2)` 并没有返回。
这种情况的概率是非常大的。另一种概率很小的情况是：先被调度的进程执行到
`accept(2)`
就发生了进程切换，而在下一次运行前，调度器启动了后一个进程，这样的话，后一个进程也将会从`select(2)`返回。

后一种情况很不容易发生， 在 `accetp(2)` 之前插入 `usleep(3)` 或 `sleep(3)`
就可以提高发生的概率了。

内核唤醒进程又不能让这个进程执行， 再次把它移动到等待队列，
造成了一定的开销浪费。`nginx` 是这样处理的：
用一个管理进程管理多个工作进程的多路复用。工作进程在`epoll_wait(2)`前向管理进程申请锁，
确保同一时刻， 多个进程在`epoll`监听的文件描述符集合的交集为空。
