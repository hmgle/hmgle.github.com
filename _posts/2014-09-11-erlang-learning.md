---
layout: post
title:  "Erlang 学习笔记"
date:   2014-09-11 20:37:53
categories: Erlang
---

## 错误总结

* erl 文件包含头文件位置不能位于 `-module(module_name).` 前面.

  那是 C 程序的做法. 虽然可以把 `-include("headfile.hrl").` 放在 `-export(...).` 前面, 但是惯例是把头文件包含放在 `-export(...).` 的后面.


## 新特性

* **maps** 是 R17 才开始支持的, 这是对 **record** 的改善. 见[Big changes to Erlang](http://joearms.github.io/2014/02/01/big-changes-to-erlang.html).
