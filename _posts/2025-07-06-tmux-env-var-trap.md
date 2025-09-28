---
layout: post
title: "tmux 环境变量陷阱"
date: 2025-07-06 11:36:08
categories: tmux
---

> **核心摘要**：`tmux server` 启动时会把当时的环境复制到自己的全局环境，并在创建新窗口（pane）时，把这份快照与会话环境合并后交给新进程。即便新 pane 内的 zsh 会重新加载 `.zshrc`，那些早已被 tmux 快照的变量仍然会存在，因此仅仅删除 `.zshrc` 里的 `export` 并不会让旧值消失。

最近在切换 Gemini CLI 认证方式时遇到了一个诡异问题：本来在 `.zshrc` 中设置了 `export GEMINI_API_KEY="xxxxx"`，运行 tmux 后，想改用 Google OAuth 认证就把这行注释掉了。结果重新开 tmux pane 时，这个环境变量居然还在！而且确认过 `.zshrc` 已经重新加载，说明问题出在 tmux 自己的环境上。

## 问题现象

```bash
# 外部 shell（正常）
$ source ~/.zshrc && printenv GEMINI_API_KEY  # 无输出

# tmux 内部（异常）
$ tmux new-window \; send-keys 'printenv GEMINI_API_KEY' Enter  # 输出：sk-xxxxxxxxxx
```

## 根本原理

Tmux 使用双层环境模型：

| 层级     | 生命周期               | 更新机制             |
| -------- | ---------------------- | -------------------- |
| 全局环境 | `tmux server` 进程周期 | 仅启动时从父进程继承 |
| 会话环境 | 单个会话周期           | 可运行时修改         |

**关键问题**：`tmux server` 启动时会把父进程的环境完整复制到自己的全局环境里；每个会话也会再维护一份会话环境。创建新 pane 时，tmux 会先把“server 全局环境 + 会话环境”合并后作为初始环境传给即将启动的 shell，随后 shell 才去读取 `.zshrc`。因此即便 `.zshrc` 里已经删除了某个 `export`，tmux 仍会把旧变量塞回去，新 shell 看到的就是包含旧值的环境。

## 解决方案

### 快速清理

```bash
# 步骤一：从 tmux server 的全局环境中移除变量，影响未来所有新窗口
tmux set-environment -gu GEMINI_API_KEY

# 步骤二：清理当前 pane 的 shell 变量，立即生效
unset GEMINI_API_KEY
```

> 可选：通过 `tmux show-environment -g GEMINI_API_KEY` 确认变量已被移除，正常情况下会提示 unknown variable。

### 彻底重置

```bash
tmux kill-server    # 重启 tmux server
tmux new-session    # 启动新会话
```

## 类似问题：PATH 重复污染

同样的机制还会导致另一个常见问题 - `PATH` 变量重复：

```bash
# 外部 shell PATH（正常）
/usr/local/bin:/usr/bin:/bin...

# tmux 内部 PATH（重复污染）
/usr/local/bin:/usr/bin:/bin.../usr/local/bin:/usr/bin:/bin...
```

这是因为每次在 tmux 内启动新 shell 时，会在已污染的 `PATH` 基础上重新执行 `.zshrc` 中的追加操作。

**解决方法**：

```bash
# 在当前 pane 去重 PATH，并把干净的值同步回 tmux server
clean_path=$(printenv PATH | tr ':' '\n' | awk '!seen[$0]++' | tr '\n' ':' | sed "s/:$//")
export PATH="$clean_path"
tmux set-environment -g PATH "$PATH"
exec zsh

# 在 ~/.zshrc 中彻底防止重复（zsh）
typeset -U path PATH
```

## 预防配置

在 `~/.tmux.conf` 中配置环境同步时，记得追加而非覆盖默认列表：

```bash
# 例：让 tmux 在新的客户端连接时同步最新的 GEMINI_API_KEY
set-option -ga update-environment "GEMINI_API_KEY"
```

`update-environment` 只会在新的客户端或会话附着时刷新列出的变量，因此重新连接前要先在外部 shell 设置好目标值；否则旧值会继续保留在 tmux 的环境里。

## 总结

1.  **`tmux server` 环境是“一次性快照”**：server 启动时的环境会被复制并沿用到所有 pane 的初始环境里。
2.  **Shell 配置只能叠加，无法回收旧值**：新 pane 会重新加载 `.zshrc`，但 tmux 会在此之前注入旧的变量，因此仅靠修改配置无法清除它们。
3.  **警惕敏感信息残留**：残留的 API Key 或 Token 可能在无意中泄露。
4.  **`PATH` 等变量易被污染**：重复追加 `PATH` 会拖慢命令查找，应在配置中做去重处理。

理解 tmux 的"环境冻存"机制，能帮我们避免很多意外的环境变量问题。

---

**参考资料**：

1. [tmux manual](https://man7.org/linux/man-pages/man1/tmux.1.html)
2. [HN 讨论](https://news.ycombinator.com/item?id=35021587)
3. [GitHub Issue](https://github.com/tmux/tmux/issues/3828)
4. [Be Careful Using tmux and Environment Variables](https://aj.codes/blog/be-careful-using-tmux-and-environment-variables/)
