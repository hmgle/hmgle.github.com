---
layout: post
title:  "一次 python segmentation fault 的调试"
date:   2014-01-04 12:58:53
categories: python segfault linux driver kernel
---

前段时间玩成语接龙，我写了个 linux [成语驱动](https://github.com/hmgle/innocent)模块(不要问为什么要在这么底层上实现这么上层的玩意儿，仅仅为了折腾一下 kernel`:)`). 应用层通过 `write`, `read` 设备节点的方式查询符合一定条件的成语。

这两天在上面用 python 写了个读写这个设备节点以判断输入是否是成语的例子，运行时出现了`segmentation fault python innocent_demo.py`的错误。python解释器出现如此严重的 **segmentation fault** 现象我还第一次见。

我为这个错误的状态建立了一个分支，要重现这个bug的话可以执行下面的步骤：

	git clone https://github.com/hmgle/innocent.git
	git checkout -b bug origin/bug
	cd innocent
	make
	sudo make install # 不安装的话可以省略， 但read/write需超级用户权限
	sudo insmod innocent.ko
	python innocent_demo.py

innocent_demo.py 只有十多行代码：

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

def is_idiom(data):
    if len(data) < 12:
        return False
    prefix = data[0:3]
    with open("/dev/innocent", "r+") as f:
        f.write('1' + prefix + '\n')
        prefix_idiom = f.readlines()
    if data[0:12] + '\n' in prefix_idiom:
        return True
    return False

t = is_idiom('逃之夭夭')
print t
t = is_idiom('一二三四')
print t
```

执行 `is_idiom('逃之夭夭')` 正常，执行 `is_idiom('一二三四')` 就挂了，python解释器直接退出， 仅输出错误信息：

	[1]    1202 segmentation fault  python innocent_demo.py

如果是 pypy 解释器的话，会输出以下错误信息：

	RPython traceback:
	  File "rpython_module_ll_os.c", line 699, in ll_os_ll_os_read
	  File "rpython_lltypesystem_rffi.c", line 2070, in str_from_buffer
	Fatal RPython error: AssertionError
	
	[1]    15811 abort (core dumped)  pypy innocent_demo.py

通过打印方式找到出错语句在第10行 `prefix_idiom = f.readlines()`. 

`readlines()`这个方法会读取文件的每一行并追加到列表尾部。以 “逃” 开头的成语只有几个，而查询 "一" 开头的成语有一千多个， `is_idiom('一二三四')` 出错而`is_idiom('逃之夭夭')` 正常，因此初步怀疑是追加到列表时 python 解释器的内存分配问题.

为了验证猜测， 我把`readlines()` 去掉，替换成 `f.read()`, 仅仅读取文件， 运行后 python解释器依然崩溃。这样就否定了内存分配的问题， 于是确定是文件io读取问题。

之前我用 `shell` 直接调用 `cat` 和 `echo` 来读写这个设备文件，表现正常， 见 [innocent_demo.sh](https://raw.github.com/hmgle/innocent/bug/innocent_demo.sh)：

```sh
#!/bin/sh

if [ ! -z "$1" ]; then
	echo "$1" > /dev/innocent
fi
cat /dev/innocent
```

为什么用 python 的 `read()` 就出错了呢？ 我用 c 写了读取这个设备文件的测试代码，发现 `open()` 之后, 分多次 `read()` 这个设备文件， 每次读取很少的字节， 程序就会崩溃。通过 `dmesg` 查看到 log 出错信息:

	segfault at 38373649 ip b7707387 sp bfaeded0 error 4 in ld-2.15.so[b76f8000+20000]

用 `strace` 跟踪到， read() 会返回一个比读取数目还要大的数，从这可以判断是驱动的问题了。

看看 innocent 驱动的 `read` 实现：

```cpp
static ssize_t innocent_read(struct file *filp, char __user *buf,
			     size_t count, loff_t *f_pos)
{
	int offset = 0;
	struct idiom_index *index;
	struct idiom_entry *entry;
	char idiom[IDIOM_LEN + 1] = { 0, };

	if (*f_pos != 0)
		return 0;
	if (prefix[0] == '\0')
		return 0;
	index = get_idiom_index(prefix, position);
	if (!index)
		return 0;
	list_for_each_entry(entry, &index->list, lists[position]) {
		memcpy(idiom, entry->idiom, IDIOM_LEN);
		idiom[IDIOM_LEN] = '\n';
		copy_to_user(buf + offset, idiom, IDIOM_LEN + 1);
		offset += IDIOM_LEN + 1;
	}
	*f_pos = offset;
	return offset;
}
```

可以看到，我没有对 `copy_to_user()` 是否成功进行判断，应用层每一次调用 `read()`, 无论读取多少字节，是否读取成功，都返回 `offset` 给应用层。这是个 `bug`!

为什么用 `cat` 的方式表现正常呢？ 用 `strace` 跟踪可以发现，`cat` 每次调用系统调用 `read()` 时，都是读取32768个字节的。这超过了驱动的返回数。假设驱动`innocent_read()` 的返回数超过 32768, 一样会出错。
而 python 的 `readlines()` 每次调用系统调用`read()`时， 是读取8k的字节，恰好 "逃" 开头的成语少于8k，而"一"开头的成语超过8k，于是开头时的崩溃现象就发生了。

那么该如何修正呢？ 我仅仅对 copy_to_user() 加了是否返回成功的判断。这样应用层不能像普通文件一样读取"innocent"了，想查询某字开头的所有成语，就只能一次性读完，不能分多次读。如果要把设备文件的读取做成普通文件那样的流形式，还需考虑许多问题。

