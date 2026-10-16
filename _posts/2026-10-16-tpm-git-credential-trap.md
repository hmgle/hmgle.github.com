---
layout: post
title: "TPM 的 git::@ 陷阱：为什么通过 TPM 安装自己的插件后无法 push"
date: 2026-10-16 09:45:00
categories: git tmux tpm
---

> **核心摘要**：TPM（Tmux Plugin Manager）安装插件时使用的 URL 是 `https://git::@github.com/用户/仓库`，里面内嵌了无效的用户名和密码，并被永久写进克隆仓库的 `.git/config`——导致 `git push` 必然认证失败，而 fetch/pull 却一切正常（公开仓库匿名可读）。本文完整记录排查过程、根因（TPM 2014 年的一个防卡死 hack）与根治工作流，并解开"昨天能发布、今天推不动"的谜团：那只是两条不同推送路径造成的错觉。

## 缘起

昨天给自己的 tmux 插件 [tmux-nexus](https://github.com/hmgle/tmux-nexus) 发布 0.8.2 版本。按日常开发工作流我把 release 交付给 codex 处理，它遵循我全局 `AGENTS.md` 中"GitHub 操作用 `gh`"的约定，顺利把分支和 tag 都推到了远端。

今天在本地改了点东西提交了一个 commit，准备手动推上去时，却撞上了权限报错：

```bash
$ git push
remote: Invalid username or token. Password authentication is not supported for Git Operations.
fatal: Authentication failed for 'https://github.com/hmgle/tmux-nexus/'
```

这就奇怪了：昨天 codex 刚推送发布成功，今天同一个仓库我自己推却直接报 401 认证失败。我把报错直接丢给 step 协助分析定位。

## 排查

step 介入后并没有去假设网络抖动，而是按顺序核对基础状态：

```bash
gh auth status           # 正常：gho_***，scopes 包含 repo
gh api user --jq .login  # 正常返回 hmgle
git ls-remote origin     # 竟然成功了，可以正常列出远程 refs
```

`gh auth status` 正常，`git ls-remote` 也能拿到远端引用，说明网络、DNS 以及 GitHub 凭据本身都没问题。问题只可能出在 git 自身的推送凭证路由或 remote 地址上。

接着检查 remote URL：

```bash
$ git remote -v
origin  https://git::@github.com/hmgle/tmux-nexus (fetch)
origin  https://git::@github.com/hmgle/tmux-nexus (push)
```

URL 里的 `git::@` 前缀显得格外扎眼。

step 模拟了 git 内部对该 URL 凭证的解析逻辑：

```bash
$ printf 'url=https://git::@github.com/hmgle/tmux-nexus\n\n' | git -c credential.helper= credential fill
protocol=https
host=github.com
path=hmgle/tmux-nexus
username=git
password=:
```

按照标准 URL 语法 `https://<username>:<password>@<host>/...`：

- 用户名被解析成了 `git`；
- 密码被解析成了一个字符 `:`。

两者都是无效凭据。因为 URL 内显式包含了 userinfo，git 在发起 HTTPS 请求时会直接把它们作为 Basic 认证发给 GitHub，自然收到 401 报错。

这里有几个极具迷惑性的地方：

1. **为什么系统里配置的 credential helper 没救场？**
   本机虽然配了 `gh auth git-credential` 作为 helper，但开启 `GIT_TRACE=1 git push --dry-run` 观察 trace 会发现，git 根本没有调用 helper 的 `get` 方法。当 URL 中已经硬编码了用户名和密码时，git 默认凭据已知，会直接跳过 helper。
2. **为什么 fetch / pull / ls-remote 一直正常？**
   tmux-nexus 是公开仓库。公开仓库的读操作不需要身份验证，GitHub 收到无效的 Basic 凭据时直接忽略，降级为匿名读取并返回数据。只有 push 这类写操作才强制鉴权，因此这个隐患一直在暗处潜伏，直到发生写操作才暴露出来。
3. **报错信息存在误导**
   git 在输出报错信息时出于安全考虑会自动隐去 URL 里的 userinfo：
   `fatal: Authentication failed for 'https://github.com/hmgle/tmux-nexus/'`
   如果只看报错日志，很容易以为 remote 地址是干干净净的。

### 为什么昨天 codex 能推上去？

如果这个 remote URL 一直是坏的，昨天 codex 是怎么把 tag 和分支推上去的？

翻看昨天的 agent 执行历史才发现，其实 codex 在第一步也遇到了同样的 401：它先尝试了 `git push -u origin fix-motion`，同样 exit 128 报错。

随后它并没有在原地反复重试，而是换了一种推送策略：

```bash
git -c credential.helper= -c 'credential.helper=!gh auth git-credential' \
    push https://github.com/hmgle/tmux-nexus.git HEAD:refs/heads/fix-motion
```

它在命令行参数里直接传入了干净的仓库 URL（没有 `git::@`），并显式绑定了 `gh auth git-credential`，绕过了本地配置的 `origin`。由于 URL 干净，git 正常向 helper 获取了有效的 GitHub Token，完成了推送。

并不是"昨天环境好好的、今天坏了"，而是昨天 codex 遇到认证失败后自主换用了一条干净的临时路径绕开了它。

## 谁写坏的 remote URL？

顺着 `.git/config` 的修改时间和 reflog 往前查：

```bash
$ git reflog --date=iso | tail -1
e657026 HEAD@{2026-07-12 17:19:32 +0800}: clone: from https://github.com/hmgle/tmux-nexus
```

时间分秒不差——`.git/config` 的修改时间与 reflog 里的克隆时间一致，畸形 URL 从克隆那一刻就写进了 config。这个目录最早是通过 TPM（Tmux Plugin Manager）自动 clone 下来的。

查看 TPM 的源码 `scripts/install_plugins.sh`，可以看到这样一段克隆逻辑：

```bash
clone_plugin() {
    local plugin="$1"
    local branch="$2"
    clone "$plugin" "$branch" ||
        clone "https://git::@github.com/$plugin" "$branch"
}
```

我的 `~/.tmux.conf` 第 87 行正是 `set -g @plugin 'hmgle/tmux-nexus'` 这样的 `user/repo` 简写。简写不是合法的 git URL，第一次 `clone "$plugin"` 必然失败，于是走 fallback，TPM 给 URL 加上 `https://git::@github.com/` 前缀重试克隆——畸形 URL 就此落进 `.git/config`，一住就是三个多月。

翻看 TPM 的 [CHANGELOG](https://github.com/tmux-plugins/tpm/blob/master/CHANGELOG.md) v1.2.1（2014 年），里面专门记录了这么改的初衷：

> It now uses the http username and password by default, like this: `https://git::@github.com/`. This prevents username and password prompt (and subsequently tmux install hanging) with old git versions. Fixes #7.

在 2014 年，`prefix + I` 在 tmux 内部非交互执行时，老版本 git 如果遇到不存在或者需要鉴权的仓库，会弹出 `Username for 'github.com':` 交互式输入，直接导致 tmux 整个界面挂起卡死。TPM 当时为了绕过这个问题，强行填了一个空的用户名和密码，逼 git 直接报错退出，避免卡死。

但实际上这个 hack 早就不需要了：从 v1.2.2（2015 年）起，TPM 就给所有的 git 命令默认加上了 `GIT_TERMINAL_PROMPT=0` 环境变量，现代 git 只要看到这个变量就会自动禁用终端交互 prompt，遇错即退。然而 2014 年遗留下来的 `git::@` fallback 却一直没人清理，成了历史包袱。

看了下本地 `~/.tmux/plugins/` 下的其他几个流行插件（如 tmux-resurrect、tmux-continuum），查看 `git remote -v` 发现它们全带着 `git::@`——大家平时只用 TPM 拉代码、从来不推代码，所以这个坑十多年来很少被人感知到。

### 上游现状

核实于 2026-10-16，TPM master（HEAD `e261deb`）里这段 fallback 代码原样存在。项目最后一次实质性代码提交停留在 2023-02，积压 101 个 open issue，官方在 #286 中的回复也自述 "relatively unmaintained"。历史上并非没人踩到：[#88（2016）](https://github.com/tmux-plugins/tpm/issues/88) 中一位用户描述了自己的插件经 TPM 安装后无法 push，得到的回复是"请使用独立的开发目录"——和本文最后的根治方案一致，然后 issue 就关闭了。十多年过去，代码依旧。

## 解决办法

### 临时修复

直接把当前仓库的 remote URL 改回标准地址：

```bash
git remote set-url origin https://github.com/hmgle/tmux-nexus
```

验证一下：

```bash
git push --dry-run
# To https://github.com/hmgle/tmux-nexus
#    28b2657..0748ddb  main -> main
```

认证路由恢复正常，随时可以真正推送。

### 避免再次踩坑的工作流

如果以后要在本机开发自己的 tmux 插件，不建议直接在 `~/.tmux/plugins/` 下 clone 和开发：一旦哪天重新通过 TPM 安装（`prefix + I`），TPM 又会重新 clone 出来一个带畸形 URL 的仓库。

更好的做法是保持开发目录独立，通过**软链接**让 TPM 识别：

```bash
# 1. 把开发目录移到专门的代码工作区
mv ~/.tmux/plugins/tmux-nexus ~/code/tmux-nexus

# 2. 软链接回 TPM 的插件目录
ln -s ~/code/tmux-nexus ~/.tmux/plugins/tmux-nexus
```

TPM 在执行 `install_plugins.sh` 时，会通过 `[ -d "$plugin_path" ]` 判断目录是否存在。由于软链接对 `-d` 生效，TPM 会将其判定为已安装，不会重新 clone 覆盖你的仓库配置。日常开发直接在 `~/code/tmux-nexus` 里用正常的 git 流程提交和推送即可。

如果不想动目录结构，另一个办法是在 `.tmux.conf` 里给这个插件写完整的 git URL——TPM 会优先按原样克隆，只有失败才走 `git::@` fallback，SSH 地址完全不受该 hack 影响：

```bash
set -g @plugin 'git@github.com:hmgle/tmux-nexus'
```

## 经验总结

- **管理自己的插件，软链是正解**：开发目录独立，软链进 `~/.tmux/plugins/`，插件管理器和开发流各管各的，谁也别想改坏谁的 URL。

---

**参考资料**：

1. [TPM CHANGELOG](https://github.com/tmux-plugins/tpm/blob/master/CHANGELOG.md)
2. [TPM issue #88：who can tell me about the url prefix 'https://git::@github.com'](https://github.com/tmux-plugins/tpm/issues/88)（2016 年一模一样的案例）
3. [TPM issue #286：GitHub rejects clones using user:pass:?@host construction](https://github.com/tmux-plugins/tpm/issues/286)（2024，open）
4. [git-credentials 文档](https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E5%87%AD%E6%9D%83%E5%AD%98%E5%82%A8)
5. [GitHub 文档：About authentication to GitHub](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/about-authentication-to-github)
