---
title: "Go-Map"
date: 2020-06-02
slug: "go map study record"
draft: false
tags:
- TECH
- Go
categories:
- TECH
---



> 参考文章: https://juejin.im/post/6844903848587296781#heading-0

## Map Struct

### 数据结构

```go
// A header for a Go map.
type hmap struct {
  // 元素个数，调用 len(map) 时，直接返回此值
	count     int
	flags     uint8
	// buckets 的对数 log_2
	B         uint8
	// overflow 的 bucket 近似数
	noverflow uint16
	// 计算 key 的哈希的时候会传入哈希函数
	hash0     uint32
  // 指向 buckets 数组，大小为 2^B
  // 如果元素个数为0，就为 nil
  // 这样理解，buckets是放在数组上的，用下标访问。 一个buckets里可能有n个bmap，是链表式的
	buckets    unsafe.Pointer
	// 扩容的时候，buckets 长度会是 oldbuckets 的两倍
	oldbuckets unsafe.Pointer
	// 指示扩容进度，小于此地址的 buckets 迁移完成
	nevacuate  uintptr
  // 溢出桶
	extra *mapextra // optional fields
}

type mapextra struct {
	// overflow[0] contains overflow buckets for hmap.buckets.
	// overflow[1] contains overflow buckets for hmap.oldbuckets.
	overflow [2]*[]*bmap
	// nextOverflow 包含空闲的 overflow bucket，这是预分配的 bucket
	nextOverflow *bmap
}

// 源码阶段的bmap，最多装 8 个 key
type bmap struct {
	tophash [bucketCnt]uint8
}

// 编译后的加料版bmap
type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr
}
```

![image-20200821140816323](https://i.loli.net/2020/08/21/itlVxOwsReyZBfp.png)



### 冷门知识点

> &nbsp;&nbsp;&nbsp;当 map 的 key 和 value 都不是指针，并且 size 都小于 128 字节的情况下，会把 bmap 标记为不含指针，这样可以避免 gc 时扫描整个 hmap。然而，bmap 还有一个 overflow 的字段，是指针类型的，如果不做处理，则会破坏了 bmap 不含指针的设想；因此这时会把 overflow 移动到 extra 字段上。

## 哈希函数的选择

> Hash 函数，有加密型和非加密型。 
>
> 加密型的一般用于加密数据、数字摘要等，典型代表就是 md5、sha1、sha256、aes256 这种。
>
> 非加密型的一般就是查找。在 map 的应用场景中，用的是查找。 
>
> 选择 hash 函数主要考察的是两点：性能、碰撞概率。
>
> 在程序启动时，会检测 cpu 是否支持 aes，如果支持，则使用 aes hash，否则使用 memhash。这是在函数 `alginit()` 中完成，位于路径：`src/runtime/alg.go` 下。

## KEY 定位

KEY 经过哈希计算后得到哈希值，共 64 个 bit 位。此时会有以下两步操作

1. 使用最后 B 个 bit 位，计算它到底要落在哪个桶时。
2. 使用高 8 位，找到此 key 在 bucket 中的位置。(最开始桶内还没有 key，新加入的 key 会找到第一个空位，放入。)
3. 图例及说明
   - 定位到桶之后，通过高8位的hash匹配槽位，如果当前bmap匹配不到，则通过overflow，到下一个bmap中匹配。
   - 源码入口: `src/runtime/map.go:395#mapaccess1`

![go-map-key-find](https://i.loli.net/2020/08/21/AKiMB7cZLgpkjqQ.jpg)

## Map Create

1. 开发编码时的写法

   ```go
   // 第一种，只初始化，不指定容量
   m := make(map[string]int)
   // 第二种，初始化且指定容量为8(值不固定，可自选)
   m := make(map[string]int, 8)
   // 第三种，只声明变量，不做初始化
   // 此时m为nil，直接向其添加元素的话会导致panic
   var m map[string]int
   ```

2. 源码实现，`src/runtime/map.go:303#makemap`

3. make说明

   - 在GO语言的编译过程中，类型检查阶段就会根据创建的类型将 `make` 替换成特定的函数，后面[生成中间代码](https://draveness.me/golang/docs/part1-prerequisite/ch02-compile/golang-ir-ssa/)的过程就不再会处理 `OMAKE` 类型的节点了，而是会根据这里生成的更加细分的操作类型进行处理。
   - Makemap 和 Makeslice 最大的区别在于，`makeslice` 函数返回的是 `Slice` 结构体，而`makemap` 函数返回的是一个指针 `*hmap`。因此，当 map 和 slice 作为函数参数时，在函数参数内部对 map 的操作会影响 map 自身，而对 slice 却不会。



## 算法解释

#### Key定位公式

1. `k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))`
2. bucket 里 key 的起始地址就是 unsafe.Pointer(b)+dataOffset。第 i 个 key 的地址就要在此基础上跨过 i 个 key 的大小；

#### Value定位公式

1. v := add(unsafe.Pointer(b), dataOffset+bucketCnt***uintptr**(t.keysize)+i***uintptr**(t.valuesize))
2. value 的地址是在所有 key 之后，因此第 i 个 value 的地址还需要加上所有 key 的偏移。

#### 遍历方式

![image-20200904143157327](https://i.loli.net/2020/09/04/VLFBQoyrHv7D94f.png)

## 装载因子

`loadFactor := count / (2^B)`, count 就是 map 的元素个数，2^B 表示 bucket 数量。

## 扩容

#### 触发时机

1. 装载因子超过阈值，源码里定义的阈值是 6.5。
2. overflow 的 bucket 数量过多
   - 当 B 小于 15，也就是 bucket 总数 2^B 小于 2^15 时，如果 overflow 的 bucket 数量超过 2^B，触发扩容；
   - 当 B >= 15，也就是 bucket 总数 2^B 大于等于 2^15，如果 overflow 的 bucket 数量超过 2^15，触发扩容。

#### 扩容代码

1. 扩容判定是在赋值方法中，`src/runtime/map.go:572#mapassign`, 赋值时，会先判定是否需要做扩容

   - ```go
     // If we hit the max load factor or we have too many overflow buckets,
     // and we're not already in the middle of growing, start growing.
     if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
     	hashGrow(t, h)
     	goto again // Growing the table invalidates everything, so try again
     }
     ```

   - `hashGrow()` 函数实际上并没有真正地“搬迁”，它只是分配好了新的 buckets，并将老的 buckets 挂到了 oldbuckets 字段上。真正搬迁 buckets 的动作在 `growWork()` 函数中，而调用 `growWork()` 函数的动作是在 mapassign 和 mapdelete 函数中。也就是插入或修改、删除 key 的时候，都会尝试进行搬迁 buckets 的工作。