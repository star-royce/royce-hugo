---
title: "Go随笔"
date: 2020-06-02
slug: "go study record"
draft: false
tags:
- TECH
- Go
categories:
- TECH
---



> 特别声明，以下内容为笔者在拜读大佬博客的过程中，随性记下的一些内容。即存在直接COPY原文的部分，也有基于原文自己做了总结的部分。
>
> 大佬博客原文请参见: https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-array-and-slice/

## 一、Slice

Go 语言中的切片有三种初始化的方式：

1. 通过下标的方式获得数组或者切片的一部分；
2. 使用字面量初始化新的切片；
3. 使用关键字 `make` 创建切片

```go
arr[0:3] or slice[0:3]
slice := []int{1, 2, 3}
slice := make([]int, 10)
```

 `[:]` 操作是创建切片最底层的一种方法。

使用如下的方式计算占用的内存：

```
内存空间 = 切片中元素大小 x 切片容量
```

在分配内存空间之前需要先确定新的切片容量，Go 语言根据切片的当前容量选择不同的策略进行扩容：

1. 如果期望容量大于当前容量的两倍就会使用期望容量；
2. 如果当前切片容量小于 1024 就会将容量翻倍；
3. 如果当前切片容量大于 1024 就会每次增加 25% 的容量，直到新容量大于期望容量；

在创建切片的过程中如果发生了以下错误就会直接导致程序触发运行时错误并崩溃：

1. 内存空间的大小发生了溢出；
2. 申请的内存大于最大可分配的内存；
3. 传入的长度小于 0 或者长度大于容量；

如果遇到了比较小的对象会直接初始化在 Go 语言调度器里面的 P 结构中，而大于 32KB 的一些对象会在堆上初始化。

`SliceHeader` 的引入能够减少切片初始化时的少量开销，这个改动能够减少 ~0.2% 的 Go 语言包大小并且能够减少 92 个 `panicindex` 的调用，占整个 Go 语言二进制的 ~3.5%。

**值传递**

单纯的从`slice`这个结构体看，我们可以通过`modify`修改存储元素的内容，但是永远修改不了`len`和`cap`，因为他们只是一个拷贝，如果要修改，那就要传递`*slice`作为参数才可以。

**For...Range的三种写法**

1. Map 举例

   ![golang-range-map](https://i.loli.net/2020/07/28/Imn2BqUoPdEajDg.png)

   

2. 注意点，遍历哈希表时会使用 [`runtime.mapiterinit`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L797-L844) 函数初始化遍历开始的元素，该函数会初始化 `hiter` 结构体中的字段，并通过 [`runtime.fastrand`](https://github.com/golang/go/blob/383b447e0da5bd1fcdc2439230b5a1d3e3402117/src/runtime/stubs.go#L99-L111) 生成一个随机数帮助我们随机选择一个桶开始遍历。Go 团队在设计哈希表的遍历时就不想让使用者依赖固定的遍历顺序，所以引入了随机数保证遍历的随机性。

   ```go
   func main() {
   	hash := map[string]int{
   		"1": 1,
   		"2": 2,
   		"3": 3,
   	}
   	for k, v := range hash {
   		println(k, v)
   	}
   }
   ```

3. 注意点，对于所有的 range 循环，Go 语言都会在编译期将原切片或者数组赋值给一个新的变量 `ha`，在赋值的过程中就发生了拷贝，所以我们遍历的切片已经不是原始的切片变量了。

   而遇到这种同时遍历索引和元素的 range 循环时，Go 语言会额外创建一个新的 `v2` 变量存储切片中的元素，**循环中使用的这个变量 v2 会在每一次迭代被重新赋值而覆盖，在赋值时也发生了拷贝**

   所以如果获取 `range` 返回变量的地址并保存到另一个数组或者哈希，结果就会是，长度正常，但是只保存了最后一个的值

   ```go
   func main() {
   	arr := []int{1, 2, 3}
   	newArr := []*int{}
   	for _, v := range arr {
   		newArr = append(newArr, &v)
   	}
   	for _, v := range newArr {
   		fmt.Println(*v)
   	}
   }
   
   $ go run main.go
   3 3 3
   ```



## 二、Map

#### 冲突解决方法

- 开放寻址法
  - 核心思想: **对数组中的元素依次探测和比较以判断目标键值对是否存在于哈希表中**, 当我们向当前哈希表写入新的数据时发生了冲突，就会将键值对写入到下一个不为空的位置
  - 数据结构: 数组
  - 性能影响关键: **装载因子**(数组中元素的数量与数组大小的比值)，随着装载因子的增加，线性探测的平均用时就会逐渐增加，这会同时影响哈希表的读写性能，当装载率超过 70% 之后，哈希表的性能就会急剧下降，而一旦装载率达到 100%，整个哈希表就会完全失效，这时查找任意元素都需要遍历数组中全部的元素

- 拉链法

  - 好处: 实现比较开放地址法稍微复杂一些，但是平均查找的长度也比较短，各个用于存储节点的内存都是动态申请的，可以节省比较多的存储空间

  - 数据结构: **链表数组**。有一些语言还会在拉链法的哈希中引入红黑树以优化性能。

  - 实现方式:

    - 哈希函数返回的哈希会帮助我们选择一个桶

      - ```go
        index := hash("Key6") % array.len
        ```

    - 选择了桶之后就可以遍历当前桶中的链表了

      - 找到键相同的键值对 —— 更新键对应的值；
      - 没有找到键相同的键值对 —— 在链表的末尾追加新键值对；

  - 拉链法的装载因子(元素数量 / 桶数量)越大，哈希的读写性能就越差，在一般情况下使用拉链法的哈希表装载因子都不会超过 1，**当哈希表的装载因子较大时就会触发哈希的扩容**，创建更多的桶来存储哈希中的元素，保证性能不会出现严重的下降

#### 溢出桶

哈希表存储的数据逐渐增多，我们会对哈希表进行扩容或者使用额外的桶存储溢出的数据，不会让单个桶中的数据超过 8 个。过多的溢出桶最终也会导致哈希的扩容。

溢出桶是在 Go 语言还使用 C 语言实现时就使用的设计[3](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/#fn:3)，由于它能够减少扩容的频率所以一直使用至今。

![image-20200713150409313](https://i.loli.net/2020/07/28/CUp43BlgEwVvGuM.png)



#### 取值语法差异

赋值语句左侧接受参数的个数会决定使用的运行时方法：

1. 当接受参数仅为一个时，会使用 [`runtime.mapaccess1`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L394-L450)，该函数仅会返回一个指向目标值的指针；
2. 当接受两个参数时，会使用 [`runtime.mapaccess2`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L452-L508)，除了返回目标值之外，它还会返回一个用于表示当前键对应的值是否存在的布尔值：

```go
v     := hash[key] // => v     := *mapaccess1(maptype, hash, &key)
v, ok := hash[key] // => v, ok := mapaccess2(maptype, hash, &key)
```



#### 扩容

以下两种情况发生时触发哈希的扩容：

1. 装载因子已经超过 6.5；翻倍扩容。
2. 哈希使用了太多溢出桶；进行等量扩容 `sameSizeGrow`，一旦哈希中出现了过多的溢出桶，它就会创建新桶保存数据，垃圾回收会清理老的溢出桶并释放内存

哈希在存储元素过多时会触发扩容操作，每次都会将桶的数量翻倍，整个扩容过程并不是原子的，而是通过 [`runtime.growWork`](https://github.com/golang/go/blob/36f30ba289e31df033d100b2adb4eaf557f05a34/src/runtime/map.go#L1104-L1113) 增量触发的，在扩容期间访问哈希表时会使用旧桶，向哈希表写入数据时会触发旧桶元素的分流；除了这种正常的扩容之外，为了解决大量写入、删除造成的内存泄漏问题，哈希引入了 `sameSizeGrow` 这一机制，在出现较多溢出桶时会对哈希进行『内存整理』减少对空间的占用。

## 三、接口

在 Go 中：实现接口的所有方法就隐式的实现了接口；

`interface{}` 类型**不是任意类型**，如果我们将类型转换成了 `interface{}` 类型，这边变量在运行期间的类型也发生了变化，获取变量类型时就会得到 `interface{}`。

|                      | 结构体实现接口 | 结构体指针实现接口 |
| -------------------- | -------------- | ------------------ |
| 结构体初始化变量     | 通过           | 不通过             |
| 结构体指针初始化变量 | 通过           | 通过               |



在如下所示的代码中，`main` 函数调用了两次 `Quack` 方法：

1. 第一次以 `Duck` 接口类型的身份调用，调用时需要经过运行时的动态派发；
2. 第二次以 `*Cat` 具体类型的身份调用，编译期就会确定调用的函数

```go
func main() {
	var c Duck = &Cat{Name: "grooming"}
	c.Quack()
	c.(*Cat).Quack()
}
```



动态派发带来的额外开销，这些额外开销在有低延时、高吞吐量需求的服务中是不能被忽视的。

|        | 直接调用 | 动态派发 |
| ------ | -------- | -------- |
| 指针   | ~3.03ns  | ~3.58ns  |
| 结构体 | ~3.09ns  | ~6.98ns  |

从上述表格我们可以看到使用结构体来实现接口带来的开销会大于使用指针实现，而动态派发在结构体上的表现非常差，这也提醒我们应当尽量避免使用结构体类型实现接口。

使用结构体带来的巨大性能差异不只是接口带来的问题，带来性能问题主要因为 Go 语言在函数调用时是传值的，动态派发的过程只是放大了参数拷贝带来的影响。

## 四、反射

Go 语言反射的三大法则，其中包括：

1. 从 `interface{}` 变量可以反射出反射对象；

   ![golang-interface-to-reflection](https://i.loli.net/2020/07/28/P7H6IOb5utVgcz2.png)

2. 从反射对象可以获取 `interface{}` 变量；

   ![golang-reflection-to-interface](https://i.loli.net/2020/07/28/I1nUsbx3FwjPJEm.png)

3. 要修改反射对象，其值必须可设置

   1. 得到的反射对象跟最开始的变量没有任何关系，所以直接对它修改会导致崩溃。

      想要修改原有的变量只能通过如下的方法

      - 调用 [`reflect.ValueOf`](https://github.com/golang/go/blob/52c4488471ed52085a29e173226b3cbd2bf22b20/src/reflect/value.go#L2316-L2328) 函数获取变量指针；

      - 调用 [`reflect.Value.Elem`](https://github.com/golang/go/blob/52c4488471ed52085a29e173226b3cbd2bf22b20/src/reflect/value.go#L788-L821) 方法获取指针指向的变量；

      - 调用 [`reflect.Value.SetInt`](https://github.com/golang/go/blob/52c4488471ed52085a29e173226b3cbd2bf22b20/src/reflect/value.go#L1600-L1616) 方法更新变量的值；

        ```go
        func main() {
        	i := 1
        	v := reflect.ValueOf(&i)
        	v.Elem().SetInt(10)
        	fmt.Println(i)
        }
        ```



## 五、defer

两大知识点

- 后调用的 defer 函数会先执行：
  - 后调用的 `defer` 函数会被追加到 Goroutine `_defer` 链表的最前面；
  - 运行 [`runtime._defer`](https://github.com/golang/go/blob/cfe3cd903f018dec3cb5997d53b1744df4e53909/src/runtime/runtime2.go#L853-L878) 时是从前到后依次执行；
- 函数的参数会被预先计算；
  - 调用 [`runtime.deferproc`](https://github.com/golang/go/blob/22d28a24c8b0d99f2ad6da5fe680fa3cfa216651/src/runtime/panic.go#L218-L258) 函数创建新的延迟调用时就会立刻拷贝函数的参数，函数的参数不会等到真正执行时计算；

使用 `defer` 时会遇到两个比较常见的问题

- `defer` 关键字的调用时机

  - `defer` 传入的函数不是在退出代码块的作用域时执行的，它只会在当前函数和方法返回之前被调用

  - ```go
    func main() {
        {
            defer fmt.Println("defer runs")
            fmt.Println("block ends")
        }
        
        fmt.Println("main ends")
    }
    
    $ go run main.go
    block ends
    main ends
    defer runs
    ```

- 多次调用 `defer` 时执行顺序是如何确定的；

  - `defer` 关键字插入时是从后向前的，而 `defer` 关键字执行是从前向后的，而这就是后调用的 `defer` 会优先执行的原因。

  - ```go
    func main() {
    	for i := 0; i < 5; i++ {
    		defer fmt.Println(i)
    	}
    }
    
    $ go run main.go
    4
    3
    2
    1
    0
    ```

- `defer` 关键字使用传值的方式传递参数时会进行预计算，导致不符合预期的结果；

  - 调用 [`runtime.deferproc`](https://github.com/golang/go/blob/22d28a24c8b0d99f2ad6da5fe680fa3cfa216651/src/runtime/panic.go#L218-L258) 函数创建新的延迟调用时就会立刻拷贝函数的参数，函数的参数不会等到真正执行时计算；

  - 想要解决这个问题的方法非常简单，我们只需要向 `defer` 关键字传入匿名函数。因为拷贝的是函数指针，所以 `time.Since(startedAt)` 会在 `main` 函数返回前被调用并打印出符合预期的结果

  - 如果希望用defer输出一个变量的最终值

    - 错误写法

      - ```go
        func main() {
        	startedAt := time.Now()
        	defer fmt.Println(time.Since(startedAt))
        	
        	time.Sleep(time.Second)
        }
        
        $ go run main.go
        0s
        ```

    - 正确写法

      - ```go
        func main() {
        	startedAt := time.Now()
        	defer func() { fmt.Println(time.Since(startedAt)) }()
        	
        	time.Sleep(time.Second)
        }
        
        $ go run main.go
        1s
        ```



## 六、panic 和 recover

特性:

1. 当程序发生崩溃时只会调用当前 Goroutine 的延迟调用函数

   ```go
   func main() {
   	defer println("in main")
   	go func() {
   		defer println("in goroutine")
   		panic("")
   	}()
   
   	time.Sleep(1 * time.Second)
   }
   
   $ go run main.go
   in goroutine
   panic:
   ...
   ```

2. `recover` 是在 `panic` 之前调用的，并不满足生效的条件。 要在 `defer` 中使用 `recover` 关键字

   ```go
   func main() {
   	defer fmt.Println("in main")
   	if err := recover(); err != nil {
   		fmt.Println(err)
   	}
   
   	panic("unknown err")
   }
   
   $ go run main.go
   in main
   panic: unknown err
   
   goroutine 1 [running]:
   main.main()
   	...
   exit status 2
   ```

3. 多次调用 `panic` 也不会影响 `defer` 函数的正常执行

   ```go
   func main() {
   	defer fmt.Println("in main")
   	defer func() {
   		defer func() {
   			panic("panic again and again")
   		}()
   		panic("panic again")
   	}()
     // panic 之后会先按顺序执行可以执行的 defer，等 defer 执行完之后才会打印 panic 里的信息
   	panic("panic once")
   }
   
   $ go run main.go
   in main
   panic: panic once
   	panic: panic again
   	panic: panic again and again
   
   goroutine 1 [running]:
   ...
   exit status 2
   ```



## 七、Make 和 New



#### 区别

- `make` 的作用是初始化内置的数据结构，并且准备了将要使用的值，只用于slice、map以及channel的初始化;

  - ```go
    // 一个包含 data、cap 和 len 的私有结构体
    slice := make([]int, 0, 100)
    // 指向 runtime.hmap 结构体
    hash := make(map[int]bool, 10)
    // 指向 runtime.hchan 结构体
    ch := make(chan int, 5)
    ```

- `new` 的作用是根据传入的类型分配一片内存空间并返回指向这片内存空间的指针，用于类型的内存分配，并且内存置为零;

  - ```go
    // 以下两种写法等价
    
    // 写法1
    i := new(int)
    
    // 写法2
    var v int
    i := &v
    ```

#### 编译期处理方式

- 在编译期间的[类型检查](https://draveness.me/golang/docs/part1-prerequisite/ch02-compile/golang-typecheck/)阶段，Go 语言就将代表 `make` 关键字的 `OMAKE` 节点根据参数类型的不同转换成了 `OMAKESLICE`、`OMAKEMAP` 和 `OMAKECHAN` 三种不同类型的节点，这些节点会调用不同的运行时函数来初始化相应的数据结构。
- 无论是直接使用 `new`，还是使用 `var` 初始化变量，它们在编译器看来就是 `ONEWOBJ` 和 `ODCL` 节点。这些节点在在[中间代码生成](https://draveness.me/golang/docs/part1-prerequisite/ch02-compile/golang-ir-ssa/)阶段都会被转换成通过 [`runtime.newobject`](https://github.com/golang/go/blob/5042317d6919d4c84557e04be35130430e8d1dd4/src/runtime/malloc.go#L1162-L1164) 函数在堆上申请内存。但是如果通过 `var` 或者 `new` 创建的变量不需要在当前作用域外生存，例如不用作为返回值返回给调用方，那么就不需要初始化在堆上。

## 八、计时器

#### 计时器维护调度

###### 1. Go 1.9 版本之前 - 全局四叉堆

- **所有的计时器由全局唯一的四叉堆维护**，存储在如下所示的结构体

  - ```go
    var timers struct {
    	lock         mutex
    	gp           *g
    	created      bool
    	sleeping     bool
    	rescheduling bool
    	sleepUntil   int64
    	waitnote     note
    	t            []*timer // 最小四叉堆
    }
    ```

- 唤醒时机:

  - 四叉堆中的计时器到期；
  - 四叉堆中加入了触发时间更早的新计时器；

- 缺点:

  - 全局四叉堆共用的互斥锁对计时器的影响非常大，计时器的各种操作都需要获取全局唯一的互斥锁，这会严重影响计时器的性能

###### 2. Go 1.10 ~ 1.13 - 分片四叉堆

- Go 1.10 将全局的四叉堆分割成了 64 个更小的四叉堆，以牺牲内存占用的代价换取性能的提升。

  - ```go
    const timersLen = 64
    
    var timers [timersLen]struct {
    	timersBucket
    }
    
    type timersBucket struct {
    	lock         mutex
    	gp           *g
    	created      bool
    	sleeping     bool
    	rescheduling bool
    	sleepUntil   int64
    	waitnote     note
    	t            []*timer
    }
    ```

- 维护方式:
  - 如果当前机器上的处理器 P 的个数超过了 64，多个处理器上的计时器就可能存储在同一个桶中
  - 每一个计时器桶都由一个运行 [`runtime.timerproc#76f4fd8`](https://github.com/golang/go/blob/76f4fd8a5251b4f63ea14a3c1e2fe2e78eb74f81/src/runtime/time.go#L195-L253) 函数的 Goroutine 处理
- 缺点:
  
  - 虽然能够降低锁的粒度，提高计时器的性能，但是 [`runtime.timerproc#76f4fd8`](https://github.com/golang/go/blob/76f4fd8a5251b4f63ea14a3c1e2fe2e78eb74f81/src/runtime/time.go#L195-L253) 造成的处理器和线程之间频繁的上下文切换却成为了影响计时器性能的首要因素

###### 3. Go 1.14 版本之后 - 每个处理器单独管理计时器并通过网络轮询器触发

- 在最新版本的实现中，计时器桶已经被移除[7](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-timer/#fn:7)，所有的计时器都以最小四叉堆的形式存储在处理器 [`runtime.p`](https://github.com/golang/go/blob/8d7be1e3c9a98191f8c900087025c5e78b73d962/src/runtime/runtime2.go#L548) 中，计时器都交由处理器的网络轮询器和调度器触发

  - ```go
    type p struct {
    	...
    	timersLock mutex // 用于保护计时器的互斥锁；
    	timers []*timer  // 存储计时器的最小四叉堆
    
    	numTimers     uint32 // 处理器中的计时器数量；
    	adjustTimers  uint32 // 处理器中处于 timerModifiedEarlier 状态的计时器数量；
    	deletedTimers uint32 // 处理器中处于 timerDeleted 状态的计时器数量；
    	...
    }
    ```

- 优势: 能够充分利用本地性、减少线上上下文的切换开销

#### 计时器数据结构

1. 私有的计时器运行时表示 - [`runtime.timer`](https://github.com/golang/go/blob/895b7c85addfffe19b66d8ca71c31799d6e55990/src/runtime/time.go#L17)

   ```go
   type timer struct {
   	pp puintptr
   
   	when     int64 // 当前计时器被唤醒的时间；
   	period   int64 // 两次被唤醒的间隔；
   	f        func(interface{}, uintptr) // 每当计时器被唤醒时都会调用的函数；
   	arg      interface{} // 计时器被唤醒时调用 f 传入的参数；
   	seq      uintptr
   	nextwhen int64 // 计时器处于 timerModifiedXX 状态时，用于设置 when 字段；
   	status   uint32 // 计时器的状态；
   }
   ```

2. 对外暴露的计时器 - [`time.Timer`](https://github.com/golang/go/blob/8d7be1e3c9a98191f8c900087025c5e78b73d962/src/time/sleep.go#L47)

   ```go
   type Timer struct {
   	C <-chan Time
   	r runtimeTimer
   }
   ```

## 九、Channel

#### 1. 底层数据结构

```go
// 底层Channel
type hchan struct {
	qcount   uint // Channel 中的元素个数；
	dataqsiz uint //  Channel 中的循环队列的长度；
	buf      unsafe.Pointer // Channel 的缓冲区数据指针；
	elemsize uint16 // 当前 Channel 能够收发的元素大小
	closed   uint32
	elemtype *_type // 当前 Channel 能够收发的元素类型
	sendx    uint  // Channel 的发送操作处理到的位置；
	recvx    uint  // Channel 的接收操作处理到的位置；
	recvq    waitq // 当前 Channel 由于缓冲区空间不足而阻塞的 Goroutine 列表，使用双向链表表示
	sendq    waitq // 当前 Channel 由于缓冲区空间不足而阻塞的 Goroutine 列表，使用双向链表表示
	lock mutex
}
// 链表元素结构
type waitq struct {
	first *sudog
	last  *sudog
}
```

#### 2. 基础规则 - FIFO

- 先从 Channel 读取数据的 Goroutine 会先接收到数据；
- 先向 Channel 发送数据的 Goroutine 会得到先发送数据的权利；

#### 3. 锁机制

Channel 是一个用于同步和通信的有锁队列。使用互斥锁解决程序中可能存在的线程竞争问题是很常见的，我们能很容易地实现有锁队列。

#### 4. 发送数据

<table><tr><td bgcolor=gray>情景分析</td></tr></table>

- 如果当前 Channel 的 `recvq` 上存在已经被阻塞的 Goroutine，那么会**直接将数据发送**给当前的 Goroutine 并将其设置成下一个运行的 Goroutine；
- 如果 Channel 存在缓冲区并且其中还有空闲的容量，我们就会**直接将数据直接存储到当前缓冲区** `sendx` 所在的位置上；
- 如果不满足上面的两种情况，就会创建一个 [`runtime.sudog`](https://github.com/golang/go/blob/895b7c85addfffe19b66d8ca71c31799d6e55990/src/runtime/runtime2.go#L342) 结构并将其加入 Channel 的 `sendq` 队列中，当前 Goroutine 也会**陷入阻塞等待**其他的协程从 Channel 接收数据；

 <table><tr><td bgcolor=gray>Goroutine调度时机</td></tr></table>

- 发送数据时发现 Channel 上存在等待接收数据的 Goroutine，立刻设置处理器的 `runnext` 属性，但是并不会立刻触发调度，而是在处理器本身下一次调度时，会优先调度此Goroutine；
- 发送数据时并没有找到接收方并且缓冲区已经满了，这时就会将自己加入 Channel 的 `sendq` 堵塞队列并调用 [`runtime.goparkunlock`](https://github.com/golang/go/blob/cfe3cd903f018dec3cb5997d53b1744df4e53909/src/runtime/proc.go#L309) 触发 Goroutine 的调度让出处理器的使用权；

#### 5. 接收数据

<table><tr><td bgcolor=gray>情景分析</td></tr></table>

- 如果 Channel 为空，那么就会直接调用 [`runtime.gopark`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/proc.go#L287) 挂起当前 Goroutine；
- 如果 Channel 已经关闭并且缓冲区没有任何数据，[`runtime.chanrecv`](https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L422) 函数会直接返回；
- 如果 Channel 的 `sendq` 队列中存在挂起的 Goroutine，就会将 `recvx` 索引所在的数据拷贝到接收变量所在的内存空间上并将 `sendq` 队列中 Goroutine 的数据拷贝到缓冲区；
- 如果 Channel 的缓冲区中包含数据就会直接读取 `recvx` 索引对应的数据；
- 在默认情况下会挂起当前的 Goroutine，将 [`runtime.sudog`](https://github.com/golang/go/blob/895b7c85addfffe19b66d8ca71c31799d6e55990/src/runtime/runtime2.go#L342) 结构加入 `recvq` 队列并陷入休眠等待调度器的唤醒；

 <table><tr><td bgcolor=gray>Goroutine调度时机</td></tr></table>

- 当 Channel 为空时；
- 当缓冲区中不存在数据并且也不存在数据的发送者时；

#### 6. 缓冲区

- 没有缓冲区的channel，必须同时存在收发动作，二者缺其一，都会造成阻塞

- 有缓冲区的channel

  - 缓冲区满之前，发送动作不会阻塞
  - 缓冲区为空的时候，接收动作会阻塞

- 代码示例

  ``` go
  // 没有缓冲区
  c := make(chan int, 0)
  
  // 有缓冲区 - 缓冲区满的时候，也会阻塞
  c := make(chan int, 10)
  ```

#### 7. Channel 关闭

**<font color=red>重点:</font>** 当 Channel 是一个空指针或者已经被关闭时，Go 语言运行时都会直接 `panic` 并抛出异常



## 十、内存管理

&emsp;&emsp;内存管理一般包含三个不同的组件，分别是用户程序（Mutator）、分配器（Allocator）和收集器（Collector）1，当用户程序申请内存时，它会通过内存分配器申请新的内存，而分配器会负责从堆中初始化相应的内存区域。

&emsp;&emsp;因为程序中的绝大多数对象的大小都在 32KB 以下，而申请的内存大小影响 Go 语言运行时分配内存的过程和开销，所以分别处理大对象和小对象有利于提高内存分配器的性能。Go 语言的内存分配器会根据申请分配的内存大小选择不同的处理逻辑，运行时根据对象的大小将对象分成微对象、小对象和大对象三种：

| 类别   | 大小          | 内存使用                                                   |
| ------ | ------------- | ---------------------------------------------------------- |
| 微对象 | `(0, 16B)`    | 先使用微型分配器，再依次尝试线程缓存、中心缓存和堆分配内存 |
| 小对象 | `[16B, 32KB]` | 依次尝试使用线程缓存、中心缓存和堆分配内存                 |
| 大对象 | `(32KB, +∞)`  | 直接在堆上分配内存                                         |

&emsp;&emsp;Go 语言的内存分配器包含内存管理单元、线程缓存、中心缓存和页堆。对应的数据结构 [`runtime.mspan`](https://github.com/golang/go/blob/921ceadd2997f2c0267455e13f909df044234805/src/runtime/mheap.go#L358)、[`runtime.mcache`](https://github.com/golang/go/blob/01d137262a713b308c4308ed5b26636895e68d89/src/runtime/mcache.go#L19)、[`runtime.mcentral`](https://github.com/golang/go/blob/8ac98e7b3fcadc497c4ca7d8637ba9578e8159be/src/runtime/mcentral.go#L20) 和 [`runtime.mheap`](https://github.com/golang/go/blob/8ac98e7b3fcadc497c4ca7d8637ba9578e8159be/src/runtime/mheap.go#L40)

 <table><tr><td bgcolor=gray>内存布局</td></tr></table>

1. Go 语言程序都会在启动初始化时，每一个处理器都会被分配一个线程缓存 [`runtime.mcache`](https://github.com/golang/go/blob/01d137262a713b308c4308ed5b26636895e68d89/src/runtime/mcache.go#L19) 用于处理微对象和小对象的分配，它们会持有内存管理单元 [`runtime.mspan`](https://github.com/golang/go/blob/921ceadd2997f2c0267455e13f909df044234805/src/runtime/mheap.go#L358)。但是，线程缓存在刚刚被初始化时是不包含 [`runtime.mspan`](https://github.com/golang/go/blob/921ceadd2997f2c0267455e13f909df044234805/src/runtime/mheap.go#L358) 的，只有当用户程序申请内存时才会从上一级组件获取新的 [`runtime.mspan`](https://github.com/golang/go/blob/921ceadd2997f2c0267455e13f909df044234805/src/runtime/mheap.go#L358) 满足内存分配的需求。

2. 每个类型的内存管理单元都会管理特定大小的对象，当内存管理单元中不存在空闲对象时，它们会从 [`runtime.mheap`](https://github.com/golang/go/blob/8ac98e7b3fcadc497c4ca7d8637ba9578e8159be/src/runtime/mheap.go#L40) 持有的 134 个中心缓存 [`runtime.mcentral`](https://github.com/golang/go/blob/8ac98e7b3fcadc497c4ca7d8637ba9578e8159be/src/runtime/mcentral.go#L20) 中获取新的内存单元。
3. 中心缓存属于全局的堆结构体 [`runtime.mheap`](https://github.com/golang/go/blob/8ac98e7b3fcadc497c4ca7d8637ba9578e8159be/src/runtime/mheap.go#L40)，它会从操作系统中申请内存。与线程缓存不同，访问中心缓存中的内存管理单元需要使用互斥锁。线程缓存会通过中心缓存的 [`runtime.mcentral.cacheSpan`](https://github.com/golang/go/blob/8ac98e7b3fcadc497c4ca7d8637ba9578e8159be/src/runtime/mcentral.go#L40) 方法获取新的内存管理单元。
4. [`runtime.mheap`](https://github.com/golang/go/blob/8ac98e7b3fcadc497c4ca7d8637ba9578e8159be/src/runtime/mheap.go#L40) 会持有 4,194,304 [`runtime.heapArena`](https://github.com/golang/go/blob/e7f9e17b7927cad7a93c5785e864799e8d9b4381/src/runtime/mheap.go#L217)，每一个 [`runtime.heapArena`](https://github.com/golang/go/blob/e7f9e17b7927cad7a93c5785e864799e8d9b4381/src/runtime/mheap.go#L217) 都会管理 64MB 的内存，单个 Go 语言程序的内存上限也就是 256TB。

 <table><tr><td bgcolor=gray>从堆中申请内存</td></tr></table>

1. 如果申请的内存比较小，获取申请内存的处理器并尝试调用 [`runtime.pageCache.alloc`](https://github.com/golang/go/blob/ebe49b2c2999a7d4128c44aed9602a69fdc53d16/src/runtime/mpagecache.go#L38) 获取内存区域的基地址和大小；
2. 如果申请的内存比较大或者线程的页缓存中内存不足，会通过 [`runtime.pageAlloc.alloc`](https://github.com/golang/go/blob/98858c438016bbafd161b502a148558987aa44d5/src/runtime/mpagealloc.go#L754) 在页堆上申请内存；
3. 如果发现页堆上的内存不足，会尝试通过 `runtime.mheap.grow` 进行扩容并重新调用 `runtime.pageAlloc.alloc`申请内存
   - 如果申请到内存，意味着扩容成功；
   - 如果没有申请到内存，意味着扩容失败，宿主机可能不存在空闲内存，运行时会直接中止当前程序；