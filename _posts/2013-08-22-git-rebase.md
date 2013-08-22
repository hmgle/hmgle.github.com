---
layout: post
title:  "git rebase 冲突解决步骤"
date:   2013-08-22 20:37:53
categories: git
---

为保证远程 `git` 仓库分支的线性特性， 可以用 `rebase` （衍合）.

假设当前分支已经 clean, 需要推送到远程仓库：

1. 本地获取远程仓库最新代码：

		git fetch origin

2. 衍合刚更新的 oringin/master 分支：

		git rebase origin/master

3. 提示冲突， 解决冲突后 add 更改的文件 foo.txt ：

		git add foo.txt

4. 无需 commit, 继续 rebase:

		git rebase --continue

5. 无冲突后衍合成功，自动回到原来的分支。最后推送到远程：

		git push origin

