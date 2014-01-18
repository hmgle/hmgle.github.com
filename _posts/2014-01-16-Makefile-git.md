---
layout: post
title:  "Git 下 C 项目的 Makefile 问题"
date:   2014-01-16 21:30:53
categories: Makefile git
---

手写 `Makefile` 时，为了自动对每一个c文件产生所依赖的头文件对应规则，一般都是添加下面这段代码:

	sinclude $(SRC:.c=.d)

	%.d: %.c
		@set -e; rm -f $@; \
			$(CC) -MM $(CPPFLAGS) $< > $@.$$$$; \
			sed 's,\(.*\)\.o[:]*,\1.o $@:,' < $@.$$$$ > $@; \
			rm -f $@.$$$$

执行 `make` 后，会先生成每一个c文件的头文件依赖规则， 保存在对应的".d"(depend)文件中。因为 ".d"文件是 `gcc` 根据文件的依赖关系自动生成的，所以用 `Git` 管理时没有必要跟踪它，在 `.gitignore` 文件中把".d"文件忽略掉. Makefile 的 clean 目标也没把它们删除。

前段时间在这样的背景下发生了这样的问题：我用`git checkout`命令把一个c仓库切换到以前的某个历史状态时， 执行`make`后，没有生成任何目标文件，屏幕也没有任何出错提示。而我清楚的记得在这个历史状态的时刻代码是可以正常编译成功的。可以执行下面的操作看到这种现象:

	git clone https://github.com/hmgle/make_test.git
	cd make_test
	make # 可以看到编译正常
	make clean
	git checkout -b develop origin/develop
	make # 编译正常
	make clean
	git checkout master
	make # 奇怪的事情发生了！

经过调试分析，才明白原来是".d"文件在作怪. 执行make clean 没有清除".d"文件，git checkout 后， ".d"文件是分支xx状态建立的，而这个分支因为加了bar.h文件，foo.c 包含了这个新加入的头文件，因此这个分支下的".d"文件会把bar.h加入foo.c的依赖文件中去，执行`git checkout master`后，里面的".d"文件是分支xx的，而master分支里面没有这个bar.h文件，因此不会编译成功。

但为什么一点错误提示都没用呢？因为Makefile 的 `sinclude` 语句会产生这样的效果：当所包含的文件不存在时不报错(这个特性和**make**的版本及平台有关, 我在**cygwin**下运行时会有提示信息`make: *** 没有规则可以创建“foo.o”需要的目标“bar.h”。 停止。`)。

从这个实例可以看到，在Makefile中的clean目标部分， 还是有理由删除".d"文件的，这样至少可以在执行一次`make clean`后，可以再次正常编译。再保守一点的话，".gitignore" 不要忽略".d"文件。如果需要自动编译系统，可以在`hooks`目录的"post-checkout"钩子文件添加删除“.d”文件的语句。
