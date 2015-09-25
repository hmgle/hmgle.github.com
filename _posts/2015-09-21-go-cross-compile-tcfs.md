---
layout: post
title:  "Go 1.5 交叉编译 TCFS-GO"
date: 2015-09-21 20:21:29
categories: Golang
---

目前 Go 最新版本 1.5 本身就已经支持各个操作系统平台及 CPU 体系的交叉编译，并且支持 cgo 的跨平台编译。搜索出来的关于 Go 交叉编译的博文大部分还是针对老版本的，目前已过时，Go 1.5 无需修改任何东西，只需指定相关的 go env 环境变量即可，具体可以参见[这篇博客](https://medium.com/@rakyll/go-1-5-cross-compilation-488092ba44ec)(墙外)，这里不再赘述。

前段时间用 Go 实现了 TCFS 这个简单的[网络文件系统的服务端](https://github.com/hmgle/tcfs-go)，于是我想把它移植到我的 Android 手机上，这样就可以借助网络在电脑挂载手机上的文件系统进行读写操作了。

一开始我想到的是使用 [go mobile](https://github.com/golang/mobile/) 来作为 Android 界面来运行 TCFS 服务，顺利编译出 APK 后安装到手机上，运行时谁知道因为权限的原因无法监听 TCP 端口，而目前仅仅使用 Go 应该配置不了 APK 的操作权限的。解决办法还得依靠 Android Studio：使用 `gomobile bind` 编译出 aar 包，里面包含了 so 库文件，界面用 Java 来写，Java 这边可以通过 `import` aar 包来使用 Go 的函数，权限可以在 manifest 文件配置。

Update: 依照这篇博文 [Writing Android apps with Go bindings](http://www.sajalkayan.com/post/android-apps-golang.html) 的介绍，我也用 Android Studio 编译出了 TCFS 的 Android 版本：

![tcfsandroid]({{ site.url }}/tcfsandroid.png)

界面还未完善，毕竟是我开发的第一个 Android 应用~ 源码托管在 https://github.com/hmgle/tcfs-android 。需要的同学可以自己编译出来，欢迎 PR 。

我的 TCFS 并不一定需要界面，我就直接编译个没有界面的命令行在 Android 上用了，进入 tcfs-go 路径后，使用下面的命令编译出 Arm 平台的程序：

```
GOOS=linux GOARCH=arm go build
```

再把编译出来的 tcfsd 复制到 Android 里面，通过终端模拟器来启动 tcfsd 服务:

![tcfsd]({{ site.url }}/images/tcfsd.png)

在 Linux 这边挂载 Android 的文件系统:

![tcfs-cli]({{ site.url }}/images/tcfs-cli.png)

这样就可以像访问本地文件一样访问 Android 里面的文件了。对比 MTP 的挂载方式，TCFS 的主要优点是不需要数据线连接，仅依赖网络连接；支持多台电脑同时挂载；缺点是受限于网络，传输速度比较慢。

这里顺便鄙视一下魅族的手机:-) 这是我使用过的问题最多的手机了，而且我碰到的不是个别问题，通过搜索发现许多人都和我一样碰到过：经常死机就不说了，关键是一死机之后就会无限重启；它的指南针从来就没有准过；以前吹嘘的最窄边框经常在边框发生误触控。。。

