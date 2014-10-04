---
layout: post
title:  "C 错误记录(二): 指针伪赋值"
date:   2014-09-25 20:41:53
categories: C
---

[上一篇](c-type.html)

我修改以前一段使用 `select(2)` 的服务端代码时, 遇到一个 segment fault.
定位到问题后, 发现是指针赋值原因.

原来的代码:

```cpp
struct xx_fd_set {
	fd_set rfds, wfds;
	fd_set _rfds, _wfds;
};

static int xx_fd_set_init(struct xx_fd_set **fd_set)
{
	*fd_set = malloc(sizeof(struct xx_fd_set));
	if (*fd_set == NULL)
		return -1;

	FD_ZERO(&(*fd_set)->rfds);
	FD_ZERO(&(*fd_set)->wfds);
	return 0;
}
```

加入 `epoll(2)` 后, 将以上函数分离到单独的文件, 不再将传入参数限定为 `struct xx_fd_set` 类型, 修改接口为:

```cpp
struct xx_server_ctx {
	int sock_fd;
	struct xx_fd_set *xx_allfd_set;
	// ...
};

static int xx_fd_set_init(struct xx_server_ctx *server)
{
	struct xx_fd_set *fd_set = server->xx_allfd_set;
	fd_set = malloc(sizeof(struct xx_fd_set));
	if (fd_set == NULL)
		return -1;

	FD_ZERO(&(fd_set)->rfds);
	FD_ZERO(&(fd_set)->wfds);
	return 0;
}
```

运行时发生 segment fault, 定位到以下代码:

```cpp
	struct xx_fd_set *fd_set = server->xx_allfd_set;
	fd_set = malloc(sizeof(struct xx_fd_set));
```

可以看到, `malloc(3)` 申请的地址并没有被 `server->xx_allfd_set` 所指向到, 而是存放在一个栈变量 `fd_set` 上. 等 `xx_fd_set_init()` 返回后, 这片申请到的内存没有被任何变量所指向, `server->xx_allfd_set` 实际上没有被赋值, 当访问`server->xx_allfd_set` 所指向的内存时, 就发生了 segment fault.

把上面的代码修改如下:

```cpp
struct xx_server_ctx {
	int sock_fd;
	struct xx_fd_set *xx_allfd_set;
	// ...
};

static int xx_fd_set_init(struct xx_server_ctx *server)
{
	struct xx_fd_set **fd_set = &server->xx_allfd_set;
	*fd_set = malloc(sizeof(struct xx_fd_set));
	if (*fd_set == NULL)
		return -1;

	FD_ZERO(&(*fd_set)->rfds);
	FD_ZERO(&(*fd_set)->wfds);
	return 0;
}
```

这样调用 `xx_fd_set_init()` 后可以正常运行. 但是, 二级指针 `fd_set` 看起来有些晦涩. 想了想是我脑子瓦特勒, 这样写岂不简单明了:

```cpp

static int xx_fd_set_init(struct xx_server_ctx *server)
{
	struct xx_fd_set *fd_set;
	fd_set = malloc(sizeof(struct xx_fd_set));
	if (fd_set == NULL)
		return -1;
	server->apidata = fd_set;
	FD_ZERO(&(fd_set)->rfds);
	FD_ZERO(&(fd_set)->wfds);
	return 0;
}
```

