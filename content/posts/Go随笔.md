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

单纯的从`slice`这个结构体看，我们可以通过`modify`修改存储元素的内容，但是永远修改不了`len`和`cap`，因为他们只是一个拷贝，如果要修改，那就要传递`*slice`作为参数才可以。

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





