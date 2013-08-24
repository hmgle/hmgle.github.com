---
layout: post
title:  "git rebase 冲突解决步骤"
date:   2013-08-22 20:37:53
categories: git
---

为维持远程 `git` 仓库分支的线性特性， 可以用 `rebase` （衍合）.

**衍合的原理**：git 找到需要衍合的两个分支的共同祖先，从这个祖先开始提取目前所在分支每次提交的差异补丁，把这些差异分别保存到临时文件，然后切换到需要衍合进来的分支，按提交顺序依次把刚才保存的差异补丁打上，成功后切换回原来的分支并快进到最新状态。

**实例步骤**：

假设当前分支工作目录状态是 clean, 需要推送到远程仓库：

1. 本地获取远程仓库最新代码：

		git fetch origin

2. 衍合刚更新的 oringin/master 分支：

		git rebase origin/master

3. 提示冲突， 解决冲突后 add 更改的文件 foo.txt ：

		git add foo.txt

4. 无需 commit, 继续 rebase:

		git rebase --continue

5. 如下一个差异补丁衍合时提示冲突，按步骤 3 处理，所有补丁打上后自动切换回原来的分支。最后推送到远程：

		git push origin


如果冲突的次数很多，衍合需要处理每次提交的冲突，比较繁琐。这时还是 `merge` 较合适。
