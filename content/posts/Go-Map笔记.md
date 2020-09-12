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



## 源码阅读方式

1. 编写一段代码，执行相应的map动作

   ```go
   package main
   
   import "fmt"
   
   func main() {
   	ageMp := make(map[string]int)
   	ageMp["qcrao"] = 18
   
   	for name, age := range ageMp {
   		fmt.Println(name, age)
   	}
   }
   ```

2. 执行go tool执行，生成汇编语言

   ```go
   go tool compile -S main.go
   ```

3. 基于汇编结果，得知实际调用的是go源码的哪个func

4. 到go源码中找到相应func实现

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

#### 触发时机/扩容方案

1. 装载因子超过阈值，源码里定义的阈值是 6.5。

   - **扩容方案:** 翻倍扩容，将 B 加 1，bucket 最大数量（2^B）直接变成原来 bucket 数量的 2 倍，会产生新老 bucket 了。由于桶的增加，此时会进行`rehash`,重新计算 key 的哈希，才能决定它到底落在哪个 bucket。某个 key 在搬迁前后 bucket 序号可能和原来相等，也可能是相比原来加上 2^B（原来的 B 值），取决于 hash 值 第 (b+1) bit 位是 0 还是 1。(<font color=red> 0 则在原桶，1则在 (2^(B+1))桶</font>)

2. overflow 的 bucket 数量过多
   - 当 B 小于 15，也就是 bucket 总数 2^B 小于 2^15 时，如果 overflow 的 bucket 数量超过 2^B，触发扩容；
   - 当 B >= 15，也就是 bucket 总数 2^B 大于等于 2^15，如果 overflow 的 bucket 数量超过 2^15，触发扩容。
   - **扩容方案:** 等量扩容，所以 B 不变。开辟一个新 bucket 空间，将老 bucket 中的元素移动到新 bucket，使得同一个 bucket 中的 key 排列地更紧密。(<font color=red> b不变，但是重新初始化了buckets </font>)
   - <font color=red>极端情况: </font> 如果插入 map 的 key 哈希都一样，就会落到同一个 bucket 里，超过 8 个就会产生 overflow bucket，结果也会造成 overflow bucket 数过多。移动元素其实解决不了问题，因为这时整个哈希表已经退化成了一个链表，操作效率变成了 `O(n)`。

3. 图解

   ![image-20200912205111282](https://img-1302326115.cos.ap-guangzhou.myqcloud.com/img/image-20200912205111282.png)

   ![image-20200912205242114](https://img-1302326115.cos.ap-guangzhou.myqcloud.com/img/image-20200912205242114.png)

   ![image-20200912205458909](https://img-1302326115.cos.ap-guangzhou.myqcloud.com/img/image-20200912205458909.png)

#### 无序Map

Go中的Map遍历是无序的，其原因有二

1. Map扩容机制为根本原因，因为扩容后，key、value的位置可能会发生变化，所以，Map是无法保障顺序的
2. 如果Map实际并没有发生过扩容，本来按本身Map的机制，也是可以有序，但是Go底层可能是为了逻辑上的统一，避免造成误解，很干脆的直接从代码层保障了Map无序遍历的结果。其实现方式为，遍历map时，并不从0开始递增遍历bucket，而是从一个随机值序号的 bucket 开始遍历，并且是从这个 bucket 的一个随机序号的 cell 开始遍历。

#### 运算符：&^

>&^，叫做`按位置 0`运算符
>
>举例：
>
>```GO
>x = 01010011
>y = 01010100
>## 如果 y bit 位为 1，那么结果 z 对应 bit 位就为 0
>## 否则 z 对应 bit 位就和 x 对应 bit 位的值相同。
>z = x &^ y = 00000011
>```

#### 扩容代码

1. 扩容判定是在赋值方法中，`src/runtime/map.go:572#mapassign`, 赋值时，会先判定是否需要做扩容

   ```go
   // If we hit the max load factor or we have too many overflow buckets,
   // and we're not already in the middle of growing, start growing.
   if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
       /**
       	hashGrow() 函数实际上并没有真正地“搬迁”，它只是分配好了新的 buckets，并将老的 buckets 挂到了 oldbuckets 字段上(申请到了新的 buckets 		空间，把相关的标志位都进行了处理)。真正搬迁 buckets 的动作在 `growWork()` 函数中。
       	而调用 `growWork()` 函数的动作是在 mapassign 和 mapdelete 函数中。也就是插入或修改、删除 key 的时候，都会尝试进行搬迁 buckets 的工作。
       */
   	hashGrow(t, h)
   	goto again // Growing the table invalidates everything, so try again
   }
   ```

2. growWork()，一共会搬迁两次。一次是当前正在使用的 bucket；另一次额外多搬迁了一次，主要是为了加快搬迁进度

   ```go
   func growWork(t *maptype, h *hmap, bucket uintptr) {
   	// 确认搬迁老的 bucket 对应正在使用的 bucket
       // 实际搬迁动作的代码在 `evacuate` 函数中
   	evacuate(t, h, bucket&h.oldbucketmask())
   
   	// 再搬迁一个 bucket，以加快搬迁进程
   	if h.growing() {
   		evacuate(t, h, h.nevacuate)
   	}
   }
   ```

   

## 遍历

>  map 遍历的核心在于理解 2 倍扩容时，老 bucket 会分裂到 2 个新 bucket 中去。而遍历操作，会按照新 bucket 的序号顺序进行，碰到老 bucket 未搬迁的情况时，要在老 bucket 中找到将来要搬迁到新 bucket 来的 key。

## 赋值

&emsp;&emsp;整体来看，流程非常得简单：对 key 计算 hash 值，根据 hash 值按照之前的流程，找到要赋值的位置（可能是插入新 key，也可能是更新老 key），对相应位置进行赋值。

&emsp;&emsp;核心还是一个双层循环，外层遍历 bucket 和它的 overflow bucket，内层遍历整个 bucket 的各个 cell。

<font color=red>划重点</font>

1. **赋值函数(`mapassign`) 实际上并没有做赋值操作，它只是定位了key的问题，进而计算出value的位置，并作为结果返回给上层调用方。**

2. 根据key类型的不同，赋值函数(`mapassign`) 还会被更针对性的函数替代。

   | key 类型 | 插入                                                         |
   | -------- | ------------------------------------------------------------ |
   | uint32   | mapassign_fast32(t *maptype, h *hmap, key uint32) unsafe.Pointer |
   | uint64   | mapassign_fast64(t *maptype, h *hmap, key uint64) unsafe.Pointer |
   | string   | mapassign_faststr(t *maptype, h *hmap, ky string) unsafe.Pointer |

3. **map 对协程是不安全的！** 赋值函数(`mapassign`)首先会检查 map 的标志位 flags。如果 flags 的写标志位此时被置 1 了，说明有其他协程在执行“写”操作，进而导致程序 panic。

   - 同时对Map做读写(一边遍历一边删除)是不被允许的。如有必要，可以使用读写锁。
   - `sync.Map` 是线程安全的 map，也可以使用。

4. **执行赋值动作时，如果 map 处在扩容的过程中，那么当 key 定位到了某个 bucket 后，需要确保这个 bucket 对应的老 bucket 完成了迁移过程。**即老 bucket 里的 key 都要迁移到新的 bucket 中来（分裂到 2 个新 bucket），才能在新的 bucket 中进行插入或者更新的操作。

5. **两个关键指针**，一个（`inserti`）指向 key 的 hash 值在 tophash 数组所处的位置，另一个(`insertk`)指向 cell 的位置（也就是 key 最终放置的地址），在循环的过程中，inserti 和 insertk 分别指向第一个找到的空闲的 cell。如果之后在 map 没有找到 key 的存在，即插入新 key。则一开始找到的空闲cell即 key 的安置地址（tophash 是 empty），而value 的位置需要“跨过” 8 个 key 的长度。

6. <font color=red>**赋值可能会触发扩容**</font>，在正式安置 key 之前，还要检查 map 的状态，看它是否需要进行扩容。如果满足扩容的条件，就主动触发一次扩容操作。假如发生了扩容，则之前查找定位key的过程，得等当前bucket扩容完毕后，重新来过。（<font color=gray>因此，为了保障处理效率，建议在可提前预估map容量的前提下，可考虑在创建map的时候提前指定好容量，尽量减少扩容次数</font>）

7. **尽量不要使用浮点数(float)作为key，**否则由于精度的问题，会导致一些诡异的问题

   - math.NaN 在map中的表现非常特殊
     - 假如用它做key，并想map中存数据了，使用m[key]的方式是拿不到结果的，而遍历可以
     - 当扩容搬迁碰到 `math.NaN()` 的 key 时，只能通过 tophash 的最低位决定分配到 X part 还是 Y part，因为NaN本身的特性决定了，`NAN != NAN`，所以多次计算它的哈希值，结果会不一样！即`hash(NAN) != hash(NAN)`
   -  float64 作为 key 的时候，先要将其转成 unit64 类型，再插入 key 中，其结果就是，`2.4` 和 `2.4000000000000000000000001` 经过 `math.Float64bits()` 函数转换后的结果是一样的。自然，二者在 map 看来，就是同一个 key 了。最终就会出现，当你存入 2.4 这个值作为key的时候，使用 2.4000、2.4000000等都是匹配不到数据的，只有`2.4` 和 `2.4000000000000000000000001`才可以找到。

## 删除

&emsp;&emsp;删除操作同样是两层循环，核心还是找到 key 的具体位置。寻找过程都是类似的，在 bucket 中挨个 cell 寻找。找到对应位置后，对 key 或者 value 进行“清零”操作，最后，将 count 值减 1，将对应位置的 tophash 值置成 `Empty`。

<font color=red>划重点</font>

1. 根据key类型的不同，删除函数(`mapdelete `) 还会被更针对性的函数替代

   | key 类型 | 删除                                              |
   | -------- | ------------------------------------------------- |
   | uint32   | mapdelete_fast32(t *maptype, h *hmap, key uint32) |
   | uint64   | mapdelete_fast64(t *maptype, h *hmap, key uint64) |
   | string   | mapdelete_faststr(t *maptype, h *hmap, ky string) |

2. **map 对协程是不安全的！**  `mapdelete` 函数首先会检查 h.flags 标志，如果发现写标位是 1，直接 panic，因为这表明有其他协程同时在进行写操作。

   - 同时对Map做读写(一边遍历一边删除)是不被允许的。如有必要，可以使用读写锁。
   - `sync.Map` 是线程安全的 map，也可以使用。

3. 如果当前map正在扩容过程中，删除操作也会触发map旧元素搬迁逻辑

