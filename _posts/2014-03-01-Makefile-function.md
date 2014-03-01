---
layout: post
title:  "Makefile 为特定目标指定选项"
date:   2014-03-01 20:37:03
categories: Makefile
---

`GNU Make` 有一个很少见的语法, 可以为特定的目标指定例外选项。
比如下面的一段代码来自 `Linux` 内核的一个 `Makefile`:

	quiet_cmd_unroll = UNROLL  $@
	      cmd_unroll = $(AWK) -f$(srctree)/$(src)/unroll.awk -vN=$(UNROLL) \
	                   < $< > $@ || ( rm -f $@ && exit 1 )
	
	ifeq ($(CONFIG_ALTIVEC),y)
	altivec_flags := -maltivec -mabi=altivec
	endif
	
	targets += int1.c
	$(obj)/int1.c:   UNROLL := 1
	$(obj)/int1.c:   $(src)/int.uc $(src)/unroll.awk FORCE
		$(call if_changed,unroll)

看上面 `$(obj)/int1.c: UNROLL := 1` 这一行，这种语法在`Makefile`里很少见。
这是什么意思呢？

它的意思是单独为 `$(obj)/int1.c` 设置 `UNROLL:=1`, 而在别的目标里， `UNROLL` 还是保留原来的值。

写个简单的例子再说明一下：

假设一个目录有两个可以生成可执行文件的源文件`foo.c`和`bar.c`, `Makefile` 是这样的：

	CFLAGS = -O2
	foo.o: CFLAGS := -g -Wall -O0
	
	target = foo bar
	
	all: $(target)
	
	foo: foo.o
	
	bar: bar.o

执行`make`后， 将这样生成目标文件:

	$ make
	cc -g -Wall -O0   -c -o foo.o foo.c
	cc   foo.o   -o foo
	cc -O2   -c -o bar.o bar.c
	cc   bar.o   -o bar

可见， `foo.o: CFLAGS := -g -Wall -O0` 这句只对 `foo.o` 设置特定的`CFLAGS`.
