---
layout: post
title:  "inetd 传递文件描述符原理"
date:  2014-10-19 21:48:48 
categories: unix linux inetd
---

## 简介

[inetd](https://en.wikipedia.org/wiki/Inetd) 号称是网络超级服务器. 它预先监听所有配置文件所定义的端口, 等到有某个客户端连接某个服务时, 再创建一个子进程运行所需的服务, 以达到降低没有客户端连接时系统负载的目的.

## 问题

我刚开始了解 inetd 时, 很好奇 inetd 是如何把自己预先准备的各种环境传递给子进程的服务的, 以使得服务能继续进行. 例如, 假设 inetd 监管一个名叫 foo 的 TCP 服务, 这个 TCP 服务本身的启动流程一定是这样的:
> 1. 创建一个 TCP 类型的套接字;
> 2. 绑定(bind)这个套接字到某块网口的一个端口上, 并在这个端口监听(listen);
> 3. 等到有客户端连接过来时, 接收(accept)这个连接, 在这个新的套接字上提供服务.

那么问题来了, 如果 inetd 预先监管这个服务, inetd 必然要替代 foo 完成以上前面两步, 等 inetd 引导 foo 启动之后, foo 是如何知道跳过前面两步直接接管传递过来的文件描述符呢, 还不能影响没有 inetd 的情况? UDP 的情况又是怎样? echo 这种没有网络连接的情况又是怎样传递文件描述符?

## 原理

为了探索背后的原理, 我在[GNU Operating System](http://ftp.gnu.org/gnu/inetutils/)下载了 inetd 的源码, 并阅读了大体的流程. 顺便说下, 获取一个 [Coreutils](https://zh.wikipedia.org/zh/GNU%E6%A0%B8%E5%BF%83%E5%B7%A5%E5%85%B7%E7%BB%84) 源码的方式是多种多样的:

- 通过包管理器: `apt-get source inetutils-inetd` # 适合于 Debian/Ubuntu
- 到 [GNU](http://www.gnu.org/software/coreutils/) 网址下载
- 下载 [busybox](http://www.busybox.net/downloads/)

inetd 根据服务的特点把服务分成大致的三类:

1. 类似 `echo`, `date` 这种本身不借助网络来通信的服务, 它通过标准输入, 标准输出, 标准出错来和用户交互
2. 服务本身通过 UDP 报文和客户端交互
3. 服务本身通过 TCP 连接和客户端交互

inetd 根据配置文件(inetd.conf)来判别服务的类型和监听的端口. 配置文件的每一行都标识了一个服务, 具体可以看[inet 手册](https://www.freebsd.org/cgi/man.cgi?query=inetd&sektion=8). 以上三种类比的服务, inetd 启动服务和传递文件描述符的方式各不相同:

1. 

2. 

3. 

## 实例

