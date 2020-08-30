---
title: "Go 垃圾回收"
date: 2020-08-30
slug: "go stw"
draft: false
tags:
- TECH
- Go
categories:
- TECH
---

>  资料参考 `https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-garbage-collector/`
>
>  现今的编程语言通常会使用手动和自动两种方式管理内存。C、C++ 以及 Rust 等编程语言使用手动的方式管理内存[2](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-garbage-collector/#fn:2)，工程师需要主动申请或者释放内存；而 Python、Ruby、Java 和 Go 等语言使用自动的内存管理系统，一般都是垃圾收集机制，不过 Objective-C 却选择了自动引用计数[3](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-garbage-collector/#fn:3)。

<table><tr><td bgcolor="895B51"><B>根本操作</B></td></tr></table>

暂停程序(Stop the world，简称STW)，然后由垃圾收集器会扫描已经分配的所有对象并回收不再使用的内存空间，清理完毕后，再恢复程序。

<table><tr><td bgcolor="895B51"><B>传统算法</B> - 标记清除（Mark-Sweep）算法</td></tr></table>

- 标记阶段 — 从根对象出发查找并标记堆中所有存活的对象；
  - 从根对象出发依次遍历对象的子对象并将从根节点可达的对象都标记成存活状态，其他的则当做垃圾
- 清除阶段 — 遍历堆中的全部对象，回收未被标记的垃圾对象并将回收的内存加入空闲链表；
  - 依次遍历堆中的所有对象，释放其中的垃圾对象并将新的空闲内存空间以链表的结构串联起来，方便内存分配器的使用。

<table><tr><td bgcolor="895B51"><B>进阶算法</B> - 三色标记算法</td></tr></table>

- 优势: 解决原始标记清除算法带来的长时间 STW（本质上还是需要STW，否则会产生悬挂指针，但是有其他技术可以补充。）
- 三色解释: 
  - 白色对象 — 潜在的垃圾，其内存可能会被垃圾收集器回收；
  - 黑色对象 — 活跃的对象，包括不存在任何引用外部指针的对象以及从根对象可达的对象；
  - 灰色对象 — 活跃的对象，因为存在指向白色对象的外部指针，垃圾收集器会扫描这些对象的子对象；
- 核心思路: 
  - 从灰色对象的集合中选择一个灰色对象并将其标记成黑色；
  - 将黑色对象指向的所有对象都标记成灰色，保证该对象和被该对象引用的对象都不会被回收；
  - 重复上述两个步骤直到对象图中不存在灰色对象；
  - 将白色对象清除掉
  - ![tri-color-mark-sweep](https://img.draveness.me/2020-03-16-15843705141814-tri-color-mark-sweep.png)
- 悬挂指针：
  - 三色标记清除算法本身是不可以并发或者增量执行的，它仍然需要 STW，否则的话，如下图，标记到最后本该是D可回收，但是在标记完毕、准备/开始清除的过程中，程序建立了从 A 对象到 D 对象的引用，则会将本来不应该被回收的对象却被回收了，这在内存管理中是非常严重的错误，我们将这种错误称为悬挂指针，即指针没有指向特定类型的合法对象，影响了内存的安全性。
  - ![tri-color-mark-sweep-and-mutator](https://img.draveness.me/2020-03-16-15843705141828-tri-color-mark-sweep-and-mutator.png)
- 屏障技术 
  - 出现的目的: 解决悬挂指针问题，保证代码对内存操作的顺序性。(在内存屏障前执行的操作一定会先于内存屏障后执行的操作)
  - 根本思路(如何避免出现悬挂指针错误?)
    - 强三色不变性 — 黑色对象不会直接指向白色对象，只会指向灰色对象或者黑色对象；
    - 弱三色不变性 — 黑色对象指向的白色对象必须包含一条从灰色对象经由多个白色对象的可达路径
  - Go语言用到的屏障技术共两种
    -  Dijkstra 提出的插入写屏障 和 Yuasa 提出的删除写屏障
    -  插入写屏障是一种相对保守的屏障技术，它会将**有存活可能的对象都标记成灰色**以满足强三色不变性。
    -   删除写屏障是在删除一个灰色对象指向某个白色对象的指针时触发，通过对此白色对象的灰着色，保证了此白色对象其下游的对象能够在这一次垃圾收集的循环中存活，避免发生悬挂指针以保证用户程序的正确性。
- 垃圾收集优化策略
  - 出现目的: 传统的垃圾收集算法会在执行期间暂停应用程序，然而很多追求实时的应用程序无法接受长时间的 STW。为了减少应用程序暂停的最长时间和垃圾收集的总暂停时间，以下两种策略应运而生。
  - 可选策略:
    - 增量垃圾收集 — 增量地标记和清除垃圾，降低应用程序暂停的最长时间；
      - 它可以将原本时间较长的暂停时间切分成多个更小的 GC 时间片，但垃圾收集开始到结束的时间更长了，即总完成时间更长。
      - 为了保证垃圾收集的正确性，我们需要在垃圾收集开始前打开写屏障，则程序还需要承担额外的计算开销。
    - 并发垃圾收集 — 利用多核的计算资源，在用户程序执行时并发标记和清除垃圾；
      - 通过开启读写屏障、**利用多核优势与用户程序并行执行**，不仅能够减少程序的最长暂停时间，还能减少整个垃圾收集阶段的时间
      - 为了保证垃圾收集的正确性，程序同样需要承担额外的计算开销。
  - **<font color=red>特别强调: </font>** 以上策略解决的是长时间STW的问题，并不能完全避免STW操作

<table><tr><td bgcolor="895B51"><B>Go 实现</B></td></tr></table>

- 策略选择

  - Go采用并发垃圾收集的策略
  - 垃圾收集器本质上多个 Goroutine的协作，通过调用方法触发状态的迁移，进而保障垃圾回收的准确处理。

- 内存操作准确性保障

  - 组合 Dijkstra 插入写屏障和 Yuasa 删除写屏障构成了**混合写屏障**，该写屏障会**将被覆盖的对象标记成灰色并在当前栈没有扫描时将新对象也标记成灰色**
  - **将创建的所有新对象都标记成黑色**，防止新分配的栈内存和堆内存中的对象被错误地回收，同时移除栈的重扫描过程

- 是否可回收的检查

  - 基本条件: 允许垃圾收集、程序没有崩溃并且没有处于垃圾收集循环
  -  [`runtime.gcTrigger.test`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/mgc.go#L1191) 方法检查通过
    - `gcTriggerHeap` — 堆内存的分配达到达控制器计算的触发堆大小；
    - `gcTriggerTime` — 如果一定时间内没有触发，就会触发新的循环;
      - 该条件由 [`runtime.forcegcperiod`](https://github.com/golang/go/blob/3093959ee10f5c28211094e784c954f6a304b9c9/src/runtime/proc.go#L4445) 变量控制，默认为 2 分钟；
    - `gcTriggerCycle` — 如果当前没有开启垃圾收集，则触发新的循环；

- 触发时机

  - #### 后台触发

    - Go 语言运行时的默认配置会在堆内存达到上一次垃圾收集的 2 倍时，触发新一轮的垃圾收集
    - 可以通过环境变量 `GOGC` 调整，在默认情况下它的值为 100，即增长 100% 的堆内存才会触发 GC。

  - #### 手动触发

    - 用户程序会通过 [`runtime.GC`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/mgc.go#L1055) 函数在程序运行期间主动通知运行时执行
      - 该方法在调用时会阻塞调用方知道当前垃圾收集循环完成
      - 在垃圾收集期间也可能会通过 STW 暂停整个程序

  - #### 申请内存

    - 内存分配器运行时会将堆上的对象按大小分成微对象、小对象和大对象三类，这三类对象的创建都可能会触发新的垃圾收集循环：
      - 当前线程的内存管理单元中不存在空闲空间时，创建微对象和小对象可能触发垃圾收集
      - 当用户程序申请分配 32KB 以上的大对象时，一定会尝试触发 垃圾收集；
    - 通过堆内存触发垃圾收集需要比较 [`runtime.mstats`](https://github.com/golang/go/blob/26154f31ad6c801d8bad5ef58df1e9263c6beec7/src/runtime/mstats.go#L24) 中的两个字段 
      - 表示垃圾收集中存活对象字节数的 `heap_live` 
        - 计算方式: 为了减少锁竞争，运行时只会在中心缓存分配或者释放内存管理单元以及在堆上分配大对象时才会更新；
      - 表示触发标记的堆内存大小的 `gc_trigger`
        - 计算方式: 在标记终止阶段调用 [`runtime.gcSetTriggerRatio`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/mgc.go#L763) 更新触发下一次垃圾收集的堆大小
    - 当内存中存活的对象字节数 `heap_live`  大于触发垃圾收集的堆大小 `gc_trigger`时，新一轮的垃圾收集就会开始

- 处理阶段共分为4步

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

- 如何暂停程序？

  - 主要使用了 [`runtime.preemptall`](https://github.com/golang/go/blob/26154f31ad6c801d8bad5ef58df1e9263c6beec7/src/runtime/proc.go#L4638) 函数来抢占所有的处理器。
  - 该函数会依次停止`当前处理器`、`等待处于系统调用的处理器`以及`获取并抢占空闲的处理器`，处理器的状态在该函数返回时都会被更新至 `_Pgcstop`，等待垃圾收集器的重新唤醒。
    - 因为程序中活跃的最大处理数为 `gomaxprocs`，所以 [`runtime.stopTheWorldWithSema`](https://github.com/golang/go/blob/3093959ee10f5c28211094e784c954f6a304b9c9/src/runtime/proc.go#L900) 在每次发现停止的处理器时都会对该变量减一，直到所有的处理器都停止运行

- 如何恢复程序

  - 程序恢复过程会使用 [`runtime.startTheWorldWithSema`](https://github.com/golang/go/blob/3093959ee10f5c28211094e784c954f6a304b9c9/src/runtime/proc.go#L976)，最终通过 [`runtime.notewakeup`](https://github.com/golang/go/blob/10e7bc994f47a71472c49f84ab782fdfe44bf22e/src/runtime/lock_sema.go#L134) 或者 [`runtime.newm`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/proc.go#L1703) 唤醒程序中的处理器
  - 实现逻辑
    - 调用 [`runtime.netpoll`](https://github.com/golang/go/blob/cfe2ab42e764d2eea3a3339aac1eaff97520baa0/src/runtime/netpoll_epoll.go#L99) 从网络轮询器中获取待处理的任务并加入全局队列；
    - 调用 [`runtime.procresize`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/proc.go#L4154) 扩容或者缩容全局的处理器；
    - 调用 [`runtime.notewakeup`](https://github.com/golang/go/blob/10e7bc994f47a71472c49f84ab782fdfe44bf22e/src/runtime/lock_sema.go#L134) 或者 [`runtime.newm`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/proc.go#L1703) 依次唤醒处理器或者为处理器创建新的线程；
    - 如果当前待处理的 Goroutine 数量过多，创建额外的处理器辅助完成任务；

- 专门处理的任务数计算

  - 后台标记任务的 CPU 利用率为 25%，所以专门任务数 = 核数 / 4
    - 主机是 4 核或者 8 核，那么垃圾收集需要 1 个或者 2 个专门处理相关任务的 Goroutine
    - 主机是 3 核或者 6 核，因为无法被 4 整除，所以这时需要 0 个或者 1 个专门处理垃圾收集的 Goroutine