---
title: "Go随笔"
date: 2020-06-02
slug: "go study record"
draft: true
tags:
- TECH
- Go
categories:
- TECH
---



> 学习文档: https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-array-and-slice/

## Slice

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

   ![golang-range-map](https://img.draveness.me/2020-01-17-15792766877639-golang-range-map.png)

   

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

## 接口

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

## 反射

Go 语言反射的三大法则，其中包括：

1. 从 `interface{}` 变量可以反射出反射对象；

   ![golang-interface-to-reflection](https://img.draveness.me/golang-interface-to-reflection.png)

2. 从反射对象可以获取 `interface{}` 变量；

   ![golang-reflection-to-interface](https://img.draveness.me/golang-reflection-to-interface.png)

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



## defer

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



## panic 和 recover

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

