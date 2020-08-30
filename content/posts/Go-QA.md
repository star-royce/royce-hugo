---
title: "Go QA"
date: 2020-08-30
slug: "go question and answer"
draft: true
tags:
- TECH
- Go
categories:
- TECH
---

## 一、 Go 内存分配

<table><tr><td bgcolor="895B51"><B>变量内存分配</B></td></tr></table>

- 无论是直接使用 `new`，还是使用 `var` 初始化变量，它们在编译器看来就是 `ONEWOBJ` 和 `ODCL` 节点。这些节点在在[中间代码生成](https://draveness.me/golang/docs/part1-prerequisite/ch02-compile/golang-ir-ssa/)阶段都会被转换成通过 [`runtime.newobject`](https://github.com/golang/go/blob/5042317d6919d4c84557e04be35130430e8d1dd4/src/runtime/malloc.go#L1162-L1164) 函数在堆上申请内存。但是如果该变量不需要在当前作用域外生存，那么就不需要初始化在堆上。
- 简单的说，编译器会做逃逸分析，当发现变量的作用域没有跑出函数范围，就可以在栈上，反之则必须分配在堆。

## 二、 Go垃圾回收机制

