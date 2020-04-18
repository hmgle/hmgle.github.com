---
layout: post
title: "gopls 魔改笔记"
date: 2019-09-08 16:46:10
categories: vim
---

gopls 作为一个符合 [LSP](https://microsoft.github.io/language-server-protocol/) 代码语言通信协议的 Go 语言部分实现，
正在被越来越多的编辑器集成进来。vim 目前也有多款插件支持 gopls 了，
比如常用的 vim-go、YouCompleteMe、vim-lsp 等。

我在使用 gopls 过程中观察到一个现象：
上述的每一个插件，在 vim 启动的时候，都会各自启动一个 gopls 进程。
如果打开多个 vim，就会启动多个 gopls。如果你同时使用 vim-go 和 YouCompleteMe 插件，
gopls 进程启动的数量还会再翻一倍。gopls 本身是个占用资源不少的东西，
对我这种机器性能还没达到过剩的电脑来说，能减少浪费还是有必要的。

放狗搜了一下相关 issue，发现还没什么讨论，插件启动各自的的 gopls 再通过 STDIO 通信目前是简单、标准的做法。
看了下 gopls 的用法，它是支持 TCP 服务模式的。那在外部启动一个 gopls 服务，
每个插件都使用 [`ch_open()`](https://vimhelp.org/channel.txt.html#ch_open%28%29) 方式连接到这个服务就可以啦。
但是对插件来说变动就大了，还可能引入[新问题](https://github.com/fatih/vim-go/issues/2421#issuecomment-529184677)。
因此，暂时不太可能有插件会启用这个节约性能的特性。

不改变插件代码的情况下，目前只能通过魔改 gopls 的方式来做，用 netcat 或 socat 作连接 TCP 服务的客户端，毕竟它们比 gopls 占用资源少太多太多了：

1. 重命名 gopls，为了让插件丧失启动 gopls 的能力：

    ```sh
    mv $GOPATH/bin/gopls{,-server}
    ```

    如果使用了 YouCompleteMe，它会启动它自己路径下第三方依赖里面的 gopls，也需要把它重命名：

    ```sh
    mv your_YouCompleteMe_DIR/third_party/ycmd/third_party/\
      go/src/golang.org/x/tools/cmd/gopls/gopls{,.bak}
    ```

2. 创建一个名字为 gopls 的脚本，它仅仅是供这些插件使用的客户端：

    ```sh
    echo '#!/bin/sh
    
    nc localhost 9877 # or socat - tcp:localhost:9877' > $GOPATH/bin/gopls
    chmod a+x $GOPATH/bin/gopls
    cp $GOPATH/bin/gopls your_YouCompleteMe_DIR/third_party/ycmd/third_party/\
      go/src/golang.org/x/tools/cmd/gopls/gopls
    ```

3. 启动 TCP 服务模式的 gopls，可以把它加入到开机启动，省的手动启动：

    ```sh
    # gopls-server 就是上面重命名后的 gopls
    gopls-server serve -listen "localhost:9877" -logfile /tmp/gopls_$RANDOM.log
    ```

4. 之后无论打开多少个编辑器，无论打开多少 go 文件，都只有一个 gopls-server 在运行了。

上面这个方法不仅仅是对于 vim 的，其他编辑器存在启动多个 gopls 的情况一样可以通过这种方式解决。
**不过，如果不在乎多给 gopls 占用一些资源的话，还是不要随便魔改，老老实实按原来方式使用就可以了。**

下次升级 gopls 后，魔改版的会被覆盖掉。

在使用过程中发现一个问题，如果使用了 YouCompleteMe，每次退出 vim 后，会发送 [exit 通知](https://microsoft.github.io/language-server-protocol/specification#exit)给 gopls-server 服务，导致 gopls-server 退出。跟踪了一下，发现是 ycmd 导致，直接把发送[退出通知](https://github.com/ycm-core/ycmd/blob/3365e2d44817d127596f59f70a6240507eb4b0bc/ycmd/completers/language_server/language_server_protocol.py#L266)的代码注释掉了:

```python
def Exit():
  # return BuildNotification( 'exit', None )
  pass
```

另一个 YouCompleteMe 的竞争者 vim-lsp 也提供了自动补全的功能，也不会出现发送 exit 通知的问题，
但使用中发现它和 gopls-server 交互太频繁了，只要是光标的位置变动，都会和 gopls 进行消息通信。
而 YouCompleteMe 只有在插入模式时才会请求服务。希望 vim-lsp 后期可以得到改进。

----------------

2020-04-18 更新:

现在 gopls 已经支持多个客户端与一个服务端后台进程进行通讯共享数据了，只要启动 gopls 时增加 "-remote=auto" 即可。现在 vim-go 和 YouCompleteMe 都已经支持添加额外的 gopls 参数了。可以更新最新版的 vim-go 和 YouCompleteMe，在 vimrc 添加下面的配置就可以尝鲜了：

```vim
" vim-go conf
let g:go_gopls_options = ['-remote=auto']

" ycmd conf
let g:ycm_gopls_args = ['-remote=auto']
```

未来也许它们会成为默认的配置，那样的话就不需要再手动添加了。

可以阅读下面的链接了解为了支持这个特性背后的故事：

- [https://github.com/golang/go/issues/34111](https://github.com/golang/go/issues/34111)
- [https://github.com/fatih/vim-go/issues/2421](https://github.com/fatih/vim-go/issues/2421)
