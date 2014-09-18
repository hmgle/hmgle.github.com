---
layout: post
title:  "Erlang 学习笔记"
date:   2014-09-11 20:37:53
categories: Erlang
---

## 基本类型

## 语法

  * `-include()` 与 `-include_lib()` 区别

## 运行环境

  ### 库

   - io 库与 io_lib 库

## 技巧
   - 在 erl shell 里查看某个模块的接口的信息和用法:
```
1> module_name:module_info().
```
因为 erl 加载的是编译过的 beam 文件, 所以查询不到具体某一个接口的实现等详细情况. 还是要看源码和doc手册.

```bash
$ erl -man module_name # 需安装 erlang-manpages
```

  - Erlang 的序列是从 **1** 开始计数的:
```
> lists:nth(1, [1, 2, 3]).
1
```

  - 清空当前进程的邮箱:
```erl
flush_buffer() ->
	receive
		AnyMessage ->
			flush_buffer()
		after 0 ->
			true
	end.
```

  - erl shell 进程崩溃后, 信箱的内容会丢失, 但变量绑定关系仍保留. 如果你在 erl shell 输入 `1 = 2.` 那么原来的 erl shell 就崩溃, 可以输入 `self()` 看到与原来的不同.

## 错误总结

  * erl 文件包含头文件位置不能位于 `-module(module_name).` 前面.

  那是 C 程序的做法. 虽然可以把 `-include("headfile.hrl").` 放在 `-export(...).` 前面, 但是惯例是把头文件包含放在 `-export(...).` 的后面.


## 新特性

  * **maps** 是 R17 才开始支持的, 这是对 **record** 的改善. 见[Big changes to Erlang](http://joearms.github.io/2014/02/01/big-changes-to-erlang.html).

## 代码升级
