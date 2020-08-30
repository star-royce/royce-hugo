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

<table><tr><td bgcolor="895B51"><B>触发时机</B></td></tr></table>

1. 基本条件
   - 允许垃圾收集、程序没有崩溃并且没有处于垃圾收集循环
2. 触发方式
   - 后台触发
   - 手动触发
   - 内存申请
     - 当前线程的内存管理单元中不存在空闲空间时，创建微对象和小对象可能触发垃圾收集
     - 当用户程序申请分配 32KB 以上的大对象时，一定会尝试触发 垃圾收集；
3. 可触发的最终条件
   - `gcTriggerHeap` — 堆内存的分配达到达控制器计算的触发堆大小；
   - `gcTriggerTime` — 如果一定时间内没有触发，就会触发新的循环，该出发条件由 [`runtime.forcegcperiod`](https://github.com/golang/go/blob/3093959ee10f5c28211094e784c954f6a304b9c9/src/runtime/proc.go#L4445) 变量控制，默认为 2 分钟；
   - `gcTriggerCycle` — 如果当前没有开启垃圾收集，则触发新的循环

<table><tr><td bgcolor="895B51"><B>算法及策略</B></td></tr></table>

1. 算法: 三色标记算法
2. 策略: 并发收集策略
   - 当主任务的回收效率不足时，启动辅助程序协助。

<table><tr><td bgcolor="895B51"><B>内存操作准确性保障 - (避免悬挂指针错误)</B></td></tr></table>

- 组合 Dijkstra 插入写屏障和 Yuasa 删除写屏障构成了**混合写屏障**，该写屏障会**将被覆盖的对象标记成灰色并在当前栈没有扫描时将新对象也标记成灰色**
- **将创建的所有新对象都标记成黑色**，防止新分配的栈内存和堆内存中的对象被错误地回收，同时移除栈的重扫描过程

<table><tr><td bgcolor="895B51"><B>回收周期</B></td></tr></table>

1. 清除终止
   - **暂停程序**
2. 标记
   - 将状态切换至 `_GCmark` 
   - 开启写屏障、用户协助程序
   - **恢复执行程序**，标记进程和用于协助的用户程序会开始并发标记内存中的对象
3. 标记终止
   - **暂停程序**，将状态切换至 `_GCmarktermination` 并关闭辅助标记的用户程序
4. 清除
   - 将状态切换至 `_GCoff`
   - 关闭写屏障
   - **恢复用户程序**
   - 后台并发执行清理

<table><tr><td bgcolor="895B51"><B>主任务数分配</B></td></tr></table>

后台标记任务的 CPU 利用率为 25%，所以专门任务数 = 核数 / 4

- 主机是 4 核或者 8 核，那么垃圾收集需要 1 个或者 2 个专门处理相关任务的 Goroutine
- 主机是 3 核或者 6 核，因为无法被 4 整除，所以这时需要 0 个或者 1 个专门处理垃圾收集的 Goroutine