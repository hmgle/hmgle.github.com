---
layout: post
title:  "搭建微信平台上的 Scheme 解释器服务"
date: 2015-09-14 21:14:47 
categories: lisp
---

我在 Scheme 微信公众号运行了一个 Scheme 解释器服务，可以对用户发送的 Scheme 表达式求值并返回结果:

![wechat]({{ site.url }}/images/schemesvr.png)

后台的 Scheme 解释器服务会对每一条微信消息都启动一个全新的初始环境，因此不同用户及不同消息互不干扰。如果要在一个环境里执行多条语句，就要把它们放在同一条微信消息，不能分开发送。比如像上面的：

```
(define x 3)
(+ x 10)
```

这个服务非常简单，运行模型如下:

```
+--------+  req: S-exp   +----------+ msgId + S-exp  +---------+  S-exp   +--------+
| wechat | ------------> | wechat   | -------------> |  Sina   | -------> | Scheme |
| client | <------------ | Platform | <------------- | App Eng | <------- | server |
+--------+ resp: result  +----------+ msgId + result +---------+  result  +--------+
```

为什么不直接在 SAE 里面运行 scheme server 呢？原因是：

1. 这个 Scheme 是 native 的，SAE 不允许运行；
2. 在我看来，这样的服务方式耦合度很低，扩展性好。假如要加入另一个 Lua 解释器 server，除修改一下服务路由之外，原有代码基本不需改动。

所有代码都已开源：[https://github.com/hmgle/wechat-scheme-server](https://github.com/hmgle/wechat-scheme-server)

怎么部署呢？

1. SAE 部分：申请、创建应用等步骤不在话下，如有困难请用搜索引擎搜索关键字`SAE 申请 创建应用`。然后克隆 https://github.com/hmgle/wechat-scheme-server/tree/master/sae 里面的代码到 SAE 代码仓库，修改 weixin.py 里面的 Scheme server 地址为自己运行 Scheme 服务的地址，修改 weixin.py 里面的 token 为自己对应的微信 token，运行后，按照微信平台说明完成与微信的握手认证。
2. Scheme server 部分：进入 `scheme-svr/yascm` 目录，按照里面的 `README.md` 编译。然后执行下面语句:

```
cd ..
nohup ./bootstrap.sh > /dev/null 2>&1 &
```

值得说明的两点是：

1. Scheme server 并没有使用 Nginx 之类的通用服务器，而是用了 `netcat` 这个工具作为网络服务接口，因此完全不需要配置。为什么会用如此简陋的工具呢？
 因为这个场景没有大并发的需求，Scheme server 仅有 SAE 这一个客户端连接，使用 `netcat` 足矣。我倾向于用最简单的方法解决问题:)
2. 并没有使用现有的 Scheme 解释器，而是用我之前实现的一个不满足 r4rs 规范简单的 Scheme 解释器轮子 [yascm](https://github.com/hmgle/yascm)。这最主要的原因是用一般的 scheme 解释器的话，相当于对用户开放了整个服务器系统，当然这可以通过 chroot 或者 Docker 容器来解决，然而我在 yascm 直接就不实现这类修改服务器系统的功能就简单得多了:) 还有一个原因是一般的解释器计算出结果使用 `printf(3)` 打印到标准输出后，需将标准输出的内容重定向到一个命名管道供 `netcat` 读取，由于默认缓存机制，如不使用 `fflush(3)`
的话，客户端是不能从 `netcat` 这里马上获得结果的。这可以通过把 `yascm` 替换为 `Racket` 或 `scheme48` 等解释器来试验，我尝试过的只有 `scm` 这个解释器是调用了 `fflush(3)` 来强行刷新 IO 的(其实前段时间这个解释器服务就是用的 `scm`)。

并不想记录用户的历史消息，因此不使用数据库。

要是没有服务器也想在局域网内的电脑运行 Scheme server，可以使用 [ngrok](https://github.com/inconshreveable/ngrok), 它通过分配一个公网地址中转请求到局域网可以使任何地方的网络都可以访问局域网内的服务。
