---
layout: post
title: "解开 tmux 在不同终端下颜色差异之谜"
date: 2024-05-12 15:23:28
categories: terminal tmux color
---

## 缘起

最近我升级了一台 macOS 下的 iTerm2 **终端模拟器**（在不引起歧义的情况下，下文简称为**终端**），从版本 3.2 升级到版本 3.4。我的主要工作方式是 tmux + Neovim，按往常一样运行 tmux 并打开 nvim，我觉察到 nvim 的颜色变了，背景色比之前的暗了许多。我使用的这套 gruvbox-material 配色主题是支持多种不同的亮度背景色的，把背景模式由原来的 'medium' 改为最亮的 'hard'，颜色看起来很好了，于是我把 nvim 颜色配置修改后推送到 git 仓库。

我的主要工作机还是另一台 Linux 主机，终端用的是 GNOME Terminal。在上面拉取完 nvim 的配置后，我又发现新的 'hard' 颜色在 tmux 上观感非常差，和在 iTerm2 上显示的颜色差别也很大。而在 iTerm2 未升级之前，使用同样的 nvim 配色，两个终端显示出来的颜色是一致的。

问题来了：为什么同样的配置，升级 iTerm2 后 tmux 的颜色发生了变化？

## 断疑

### 前置知识

首先需要梳理一下，在终端运行 tmux，又在 tmux 打开 Neovim，Neovim 的颜色信息是如何发送到终端的：大致的过程是 Neovim 的输出会经过 tmux 捕获处理，然后 tmux 再把处理后的内容输出给终端。

这些颜色信息在终端是使用 [**ANSI 转义序列**（ANSI escape code）](https://en.wikipedia.org/wiki/ANSI_escape_code)来表示的。

一般的现代 vim 配色主题会有两套颜色方案：256 色的 "cterm" 以及 24 位 RGB 格式的 "gui"，vim 默认是选择其中的 "cterm" 方案，除非是开启了 'termguicolors' 属性。
我的 Neovim 配置是开启了 'termguicolors'，所以它输出给 tmux 的是 RGB 颜色格式的 ANSI 转义序列。

tmux 对输入的这些 ANSI 转义序列并不会直接发给终端，而是根据推断出来的终端是否具备 24 位颜色渲染特性，再决定是否把这些 RGB 颜色转换为 256 色。如果 tmux 推断出终端只能渲染 256 色，那么它输出给终端的颜色信息就是经过转换后的 256 色。

有了上面的知识储备之后，就可以猜测上面终端在同样的配置下渲染出不同的颜色很可能是因为 tmux 推断终端的渲染能力不准确导致了 RGB 颜色被转换为 256 色输出给终端。

可以这样验证 tmux 识别的终端是否具备 24 位颜色渲染能力：使用 `tmux -vvv` 生成 tmux server 的启动日志，过滤最后一条 "RGBCOLOURS flag" 日志内容，看结果为 0 还是 1。或者执行 `tmux info | grep -i rgb` 观察结果，有 RGB 标志位或者有 setrgbb 与 setrgbf 字符串（这两个字符串代表了用 RGB 参数设置背景和前景色的能力）即表明 tmux 在 24 位色模式下运行。

经验证，新版（3.4）的 iTerm2 正确识别出 RGBCOLOURS flag 为 1，而旧版（3.2）iTerm2 及 GNOME Terminal 识别出的 RGBCOLOURS flag 为 0。也就是说，我之前使用的是 256 色模式的 tmux！尽管我的这些终端都支持 24 位 RGB 颜色。

### 配置的错误

我检查了下我的 tmux 配置：

```
set -as terminal-features ",gnome*:RGB"
```

想起了当时是看到了 tmux 官方文档的[How do I use RGB colour?](https://github.com/tmux/tmux/wiki/FAQ#how-do-i-use-rgb-colour)，直接拷贝过来的：

```diff
-set -g default-terminal "xterm"
-set -ga terminal-overrides ",*256col*:Tc"
+set -g default-terminal "xterm-256color"
+set -as terminal-features ",gnome*:RGB"
```

现在明白了，这句配置的意思是只有环境变量 `$TERM` 匹配上了 `gnome*` 这个正则表达式时 tmux 才会为该添加 RGB 特性，而我的终端的`$TERM` 环境变量是 "xterm-256color"，当然就匹配不上这条规则了。需要根据具体的 `$TERM` 来修改这条规则，把上面的 "gnome*" 改为 "xterm*" 就能匹配上了：

```diff
-set -as terminal-features ",gnome*:RGB"
+set -as terminal-features ",xterm*:RGB"
```

### 为什么新版 iTerm2 可以无视配置规则而被 tmux 加上 RGB 颜色特性呢？

浏览 tmux 代码发现，tmux 会为部分终端添加一些特性：

```c
void
tty_default_features(int *feat, const char *name, u_int version)
{
	static struct {
		const char	*name;
		u_int		 version;
		const char	*features;
	} table[] = {
#define TTY_FEATURES_BASE_MODERN_XTERM \
	"256,RGB,bpaste,clipboard,mouse,strikethrough,title"
                ...
		{ .name = "iTerm2",
		  .features = TTY_FEATURES_BASE_MODERN_XTERM
			      ",cstyle,extkeys,margins,usstyle,sync,osc7"
		},
                ...
	};
        ...
}
...
/* Add terminal features. */
tty_default_features(&c->term_features, "iTerm2", 0);
...
```

只要识别出是 iTerm2，tmux 就会添加 RGB 颜色特性。问题的关键是 tmux 如何识别出新版 iTerm2 的 APP 名称的？原来 tmux 会向终端发送一个扩展的获取终端信息 ANSI 转移序列 `CSI > q`，支持这个 `CSI > q` 查询的终端会返回自己的 APP 名称和版本号，这也是 3.4 版本的 iTerm2 添加的功能：[Extended Device Attributes](https://iterm2.com/documentation-escape-codes.html)，而我之前用的 3.2 版本并不会响应这个查询，因此也无法在 tmux 自动开启 RGB 颜色特性了。

让运行在终端的程序识别出终端的名称和版本号并不是一件简单的事情，如果终端都能支持这个 `CSI > q` 查询当然是最好的了，但很多老牌的或者旧版本的终端都是不支持这个扩展的 ANSI 转义序列查询的。可以这样验证：在终端运行 `printf "\x1B[>q"` 看会不会打印出 APP 名称和版本号。

### 获取终端特性的另一种手段：terminfo

后来我又尝试了在另一个终端 kitty 上运行 tmux，出现了一个有趣的现象：即使不为 tmux 配置任何的 RGB 特性（即默认的 tmux 配置，不添加 `terminal-overrides` 或 `terminal-features` RGB 特性）通过 brew 安装的 tmux，可以自动为 Kitty 开启 RGB 色彩特性，而我自己编译出来的 tmux，却无法自动开启这个特性。

再一次的探索发现了背后的原因：tmux 会通过 `setupterm()` 和 `tigetstr()` 等函数查询 [terminfo](https://linux.die.net/man/5/terminfo) 来获取终端的特性，如果查询出来的 terminfo 数据包含了 "setrgbb" 和 "setrgbf"，则为该终端开启 RGB 特性。关键代码如下：

```c
	/* Update the RGB flag if the terminal has RGB colours. */
	if (tty_term_has(term, TTYC_SETRGBF) &&
	    tty_term_has(term, TTYC_SETRGBB))
		term->flags |= TERM_RGBCOLOURS;
	else
		term->flags &= ~TERM_RGBCOLOURS;
	log_debug("RGBCOLOURS flag is %d", !!(term->flags & TERM_RGBCOLOURS));
```

kitty 和其他终端不一样的是，它维护了自己的 terminfo 数据，默认配置下的 kitty 启动时会设置环境变量 "$TERM" 为 "xterm-kitty"，"$TERMINFO" 被设置为 kitty terminfo 数据的目录。它自己的 terminfo 包含了一些 kitty 特有的特性，其中就包括了 "setrgbb" 和 "setrgbf"。tmux 正是根据这些信息确定了当前的终端可以开启 RGB 特性。

为啥我自己编译的 tmux 不能把 "setrgbb" 和 "setrgbf" 查询出来呢？

用 `otool -L` （相当于 Linux 的 `ldd` 命令）对比我自己编译的 tmux 和 brew 安装的 tmux 差异可以看出，它们链接的库不一样：

```
$ otool -L /usr/local/Cellar/tmux/3.4_1/bin/tmux
        ...
        /usr/local/opt/ncurses/lib/libncursesw.6.dylib (compatibility version 6.0.0, current version 6.0.0)
        ...
$ otool -L 自己编译的tmux/tmux
        ...
        /usr/lib/libncurses.5.4.dylib (compatibility version 5.4.0, current version 5.4.0)
        ...
```

tmux 里面查询 terminfo 的`setupterm()` 和 `tigetstr()` 函数正是链接到上面的库的，这个版本的 libncurses 会直接忽略 "setrgbb" 和 "setrgbf"。从搜索的结果来看，老版本的 ncurses 的开发者似乎对这两个非传统的变量不太感冒：[Support 24-bit terminals](https://lists.gnu.org/archive/html/bug-ncurses/2016-08/msg00035.html)、[一些用户的评论](https://gist.github.com/XVilka/8346728?permalink_comment_id=2255317#gistcomment-2255317)。
通过 brew 安装的 tmux 是依赖 libncursesw 的， 它可以返回这两个变量的字符串值。

另外，kitty 这种使用自己的 terminfo 数据的方式的确可以为外部程序提供更多的关于自己特性的信息，但也并非完美。比如：我在 kitty 上 ssh 一台 Linux 主机后，再运行 tmux 时就报错了："missing or unsuitable terminal: xterm-kitty"，这是因为 ssh 会直接继承之前的 `TERM=xterm-kitty` 的环境变量，在远程主机运行 tmux 查询 `$TERM` 的 terminfo 时，远程主机是没有安装 kitty 的，所以会缺失 xterm-kitty 的相关信息。这里有关于 kitty 这部分的相关讨论：

- https://news.ycombinator.com/item?id=24643938
- [Please submit xterm-kitty terminfo to ncurses database](https://github.com/kovidgoyal/kitty/issues/879)
- [Can we talk about "xterm-kitty"?](https://github.com/kovidgoyal/kitty/discussions/3873)

## 结语

要准确地判断出各色各样的终端模拟器的色彩特性并非是一件简单的事情，需要兼容不同标准，还要兼顾许多历史遗留问题。这也正是 tmux 提供了给用户配置 `terminal-features` 和 `terminal-overrides` 的原因。也许，可以像 `CSI > q` 这样，增加 ANSI 转移序列用来查询终端特性，让终端自己回答。

## 相关链接

- [Terminal Colors](https://github.com/termstandard/colors?tab=readme-ov-file)
- [True color not wroking](https://github.com/tmux/tmux/issues/2783)
- [True color doesn't work in kitty anymore with 3.2-rc](https://github.com/tmux/tmux/issues/2418)
