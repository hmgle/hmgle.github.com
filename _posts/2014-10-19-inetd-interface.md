---
layout: post
title:  "inetd 传递文件描述符原理"
date:  2014-10-19 21:48:48 
categories: unix linux inetd
---

## 简介

[inetd](https://en.wikipedia.org/wiki/Inetd) 号称是网络超级服务器. 它预先监听所有配置文件所定义的端口, 等到有某个客户端连接某个服务时, 再创建一个子进程运行所需的服务, 以达到降低没有客户端连接时系统负载的目的.

inetd 的现代版为 xinetd, 它增加了许多安全的功能.

## 问题

我刚开始了解 inetd 时, 很好奇 inetd 是如何把自己预先准备的各种环境传递给子进程的服务的, 以使得服务能继续进行. 例如, 假设 inetd 监管一个名叫 foo 的 TCP 服务, 这个 TCP 服务本身的启动流程一定是这样的:
> 1. 创建一个 TCP 类型的套接字;
> 2. 绑定(bind)这个套接字到某块网口的一个端口上, 并在这个端口监听(listen);
> 3. 等到有客户端连接过来时, 接收(accept)这个连接, 在这个新的套接字上提供服务.

那么问题来了, 如果 inetd 预先监管这个服务, inetd 必然要替代 foo 完成以上前面两步, 等 inetd 引导 foo 启动之后, foo 是如何知道跳过前面两步直接接管传递过来的文件描述符呢, 还不能影响在没有 inetd 时由服务本身独立启动的情况? UDP 的情况又是怎样? echo 这种没有网络连接的情况又是怎样传递文件描述符?

## 原理

为了探索背后的原理, 我在[GNU Operating System](http://ftp.gnu.org/gnu/inetutils/)下载了 inetd 的源码, 并阅读了大体的流程. 顺便说下, 获取一个 [Coreutils](https://zh.wikipedia.org/zh/GNU%E6%A0%B8%E5%BF%83%E5%B7%A5%E5%85%B7%E7%BB%84) 源码的方式是多种多样的:

- 通过包管理器: `apt-get source inetutils-inetd` # 适合于 Debian/Ubuntu
- 到 [GNU](http://www.gnu.org/software/coreutils/) 网址下载
- 下载 [busybox](http://www.busybox.net/downloads/)

inetd 根据服务的特点把服务分成大致的三类:

1. 类似 `echo`, `date` 这种本身不借助网络来通信的服务, 它通过标准输入, 标准输出, 标准出错来和用户交互
2. 服务本身通过 TCP 连接和客户端交互
3. 服务本身通过 UDP 报文和客户端交互

inetd 根据配置文件(inetd.conf)来判别服务的类型和监听的端口. 配置文件的每一行都标识了一个服务, 具体可以看[inet 手册](https://www.freebsd.org/cgi/man.cgi?query=inetd&sektion=8). 以上三种类别的服务, inetd 启动服务和传递文件描述符的方式各不相同:

### 1. 仅用 `STDIN_FILENO`, `STDOUT_FILENO` 和 `STDERR_FILENO` 与用户交互的服务

  比如 `date`, 假如配置文件为:

```bash
tcpmux stream  tcp nowait root /bin/date  date
```

  inetd 会在 tcpmux 这个服务对应的 TCP 端口监听连接, 等到有客户端连接(`connect(2)`)过来后, inetd 接受(`accept(2)`) 并创建一个新套接字的文件描述符, 再 `fork(2)` 出子进程, 子进程会继承这个文件描述符, 只要将这个文件描述符 `dup2(2)` 到 `STDIN_FILENO`,  `STDOUT_FILENO` 和 `STDERR_FILENO` 就可以了. 通过 `execv(3)` 启动原始的服务, 这样, `date` 读写stdio设备实际上就是向连接客户端的套接字进行读写了.

  可以用 `netcat` 这样模拟客户端:

```console
$ nc localhost 1
2014年 10月 20日 星期一 17:11:57 CST
```

  unix 上的许多工具都遵循这这样的传统: 默认从 `STDIN_FILENO` 读入, 写出到 `STDOUT_FILENO`, 出错信息输出到 `STDERR_FILENO`. 例如 `dd`, `cat`, 这意味着它们可以通过 inetd 管理的的方式提供网络服务. 对于常见的服务, inetd 本身实现了内置的服务, 这可以在配置文件中通过 `internal` 选项指明使用内置的服务.

### 2. 服务本身通过 TCP 连接与客户端交互

  这种情况需要符合一定的规则才可以由 inetd 顺利启动并传递文件描述符的. 而且 inetd 的配置行必须要设置为 "wait" 方式, 以使得新的客户端连接由子进程的服务接管. 

  这里有一个关键点: 就是 inetd 在什么时候 `fork(2)` 并传入文件描述符, 客户端只能 `connect(2)` 一次, 到底是由 inetd `accept(2)` 还是服务本身 `accept(2)`? 客户端由 inetd 启动时到底要省略 TCP 服务启动流程的哪些步骤?

  TCP 服务中 `accept(2)` 一般是位于 `listen(2)` 之后的无限循环中的, 设想一下, 假设 inetd 设计为将 `accept(2)` 之后的环境传递给启动的服务, 那么这个服务必须跳过前面的 `listen(2)` 和 `accept(2)` 环节, 它就像前面的 `echo` `date` 之类的服务一样, 失去了为多个客户端提供服务的能力了.
  因此, inetd 是这样监听并启动这类 tcp wait 方式的服务的:

  1. 监听(`listen(2)`)服务对应的 TCP 端口, 把这个文件描述符加入 fd_set 集合
  2. 在循环体内 `select(2)` 等待这个文件可读, 然后 `fork(2)` 出子进程, 再利用 `dup2(2)` 复制监听端口的文件描述符到标准读写设备,  子进程继承的是这个监听端口的文件描述符, 最后在子进程里启动服务

  对比第一类服务, 唯一区别就是第一类中 inetd 多了 `accept(2)` 这个环节.

  被 inetd 接管的服务需要根据以上的特点改造一下才能既能独立启动, 又能被 inetd 启动. 我写一个简单的例子, 假设这个服务为 foo:

  inetd.conf 文件需要加入一行这样的配置:

```bash
dcap stream tcp wait root /home/gle/inetd_test/foo foo # 路径根据实际情况自己修改
```

  foo.c 代码:

```cpp
#include <unistd.h>
#include <stdlib.h>
#include <stdint.h>
#include <stdio.h>
#include <string.h>
#include <netdb.h>
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <sys/mman.h>
#include <sys/select.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/stat.h>
#include <unistd.h>
#include <fcntl.h>
#include <arpa/inet.h>
#include <sys/time.h>
#include <errno.h>

static int bind_server(int server_s, char *server_ip, uint16_t server_port)
{
	struct sockaddr_in server_sockaddr;

	memset(&server_sockaddr, 0, sizeof server_sockaddr);
	server_sockaddr.sin_family = AF_INET;
	if (server_ip != NULL)
		inet_aton(server_ip, &server_sockaddr.sin_addr);
	else
		server_sockaddr.sin_addr.s_addr = htonl(INADDR_ANY);
	server_sockaddr.sin_port = htons(server_port);
	return bind(server_s, (struct sockaddr *) &server_sockaddr,
			sizeof(server_sockaddr));
}

int main(int argc, char **argv)
{
	int ser_fd;
	struct sockaddr saddr;
	socklen_t slen = sizeof(saddr);
	int port = 22125; /* dcap 服务的端口 */

	char *buf = "hello!\n";
	ser_fd = STDIN_FILENO;
	if (getsockname(ser_fd, &saddr, &slen) < 0) {
		fprintf(stderr, "getsockname return -1\n");
		if (errno == ENOTSOCK) {
			ser_fd = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
			if (ser_fd == -1)
				exit(1);
		} else {
			fprintf(stderr, "err: errno is %s\n", strerror(errno));
			exit(1);
		}
		int sock_opt = 1;
		if (setsockopt(ser_fd, SOL_SOCKET, SO_REUSEADDR,
				(void *)&sock_opt, sizeof(sock_opt)) == -1)
			exit(1);
		if (bind_server(ser_fd, NULL, port) == -1)
			exit(1);
	}
	if (listen(ser_fd, 100) == -1) {
		fprintf(stderr, "err: errno is %s\n", strerror(errno));
		exit(1);
	}

	fd_set rset;
	int ret;
	int connfd;
	for (;;) {
		FD_ZERO(&rset);
		FD_SET(ser_fd, &rset);
		ret = select(ser_fd + 1, &rset, NULL, NULL, NULL);
		if (ret < 0) {
			exit(1);
		} else if (ret > 0 && FD_ISSET(ser_fd, &rset)) {
			connfd = accept(ser_fd, NULL, 0);
			if (connfd < 0) {
				perror("accept err");
				exit(1);
			}
			write(connfd, buf, 7);
			break;
		}
	}
	return 0;
}
```

  客户端通过 netcat 来模拟:

```console
$ nc 127.0.0.1 22125
hello!
```

### 3. UDP 服务

  对于无连接的 UDP, 同样需要在 inetd 的配置文件中设置 wait 选项, 传递套接字的文件描述符方式与前文一致, 不再赘述.
