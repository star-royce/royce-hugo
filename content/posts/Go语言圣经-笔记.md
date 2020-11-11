---
title: "Go语言圣经-笔记"
date: 2020-10-22
slug: "go language notes"
draft: false
tags:
- Tech
- SelfLearning
categories:
- TECH
---



[Go 语言圣经][https://books.studygolang.com/gopl-zh/ch0/ch0-01.html] 拜读笔记

## fmt 格式

![image-20201109154438759](https://i.loli.net/2020/11/09/qhdYfsXvx7rgmAz.png)

## 指针操作(&与*)

- &操作符可以返回一个变量的内存地址

- *操作符可以获取指针指向的变量内容

## 标准库包查询及文档阅读

- 可以在 https://golang.org/pkg 和 [https://godoc.org](https://godoc.org/) 中找到标准库和社区写的package
- godoc这个工具可以让你直接在本地命令行阅读标准库的文档。`go doc http.ListenAndServe`

## Go命名规范

1. 大小写敏感。heapSort和Heapsort是两个不同的名字。
2. 大小写规定了变量的作用域。一个名字是大写字母开头的，才可以被外部的包访问。
3. 包本身的名字一般总是用小写字母。
4. Go语言的风格是尽量使用短小的名字，对于局部变量尤其是这样。如果一个名字的作用域比较大，生命周期也比较长，那么用长的名字才会更有意义。
5. Go语言程序员推荐使用 **驼峰式** 命名

## 如何知道一个变量是何时可以被回收的呢？

基本的实现思路是，从每个包级的变量和每个当前运行函数的每一个局部变量开始，通过指针或引用的访问路径遍历，是否可以找到该变量。如果不存在这样的访问路径，那么说明该变量是不可达的，也就是说它是否存在并不会影响程序后续的计算结果。

## 在栈上还是在堆上分配局部变量的存储空间？

<font color=red>这个选择并不是由用var还是new声明变量的方式决定的。</font>

1. 如果发生了逃逸，则一定是在堆上分配内存。会引起的语法主要有以下几种：
   - 使用指针作为切片值
   - 发送指针到channel
   - 使用interface内存，且通过interface来调用方法
2. 如果没发生逃逸，进一步考虑所需内存大小
   - 根据对象的大小，分成三类：
     - 微对象（小于等于16B），使用mcache的tiny分配器分配。会使用线程缓存上的微分配器提高微对象分配的性能。
     - 小对象（大于16B，小于等于32KB），首先计算对象的规格大小，然后使用mcache中相应规格大小的mspan分配；从线程缓存、中心缓存或者堆中获取内存管理单元并从内存管理单元找到空闲的内存空间；
     - 大对象（大于32KB），32KB 的对象，直接从mheap上分配，按照 8KB 的倍数为对象在堆上申请内存。

## 类型强转

类型转换不会改变值本身，但是会使它们的语义发生变化。<font color=red>但前提是两个类型的底层基础类型相同，或者是两者都是指向相同底层结构的指针类型！</font>否则，可能改变值的表现。例如，将一个浮点数转为整数将丢弃小数部分。

```go
package tempconv

import "fmt"

type Celsius float64    // 摄氏温度
type Fahrenheit float64 // 华氏温度

const (
    AbsoluteZeroC Celsius = -273.15 // 绝对零度
    FreezingC     Celsius = 0       // 结冰点温度
    BoilingC      Celsius = 100     // 沸水温度
)

func CToF(c Celsius) Fahrenheit { return Fahrenheit(c*9/5 + 32) }

func FToC(f Fahrenheit) Celsius { return Celsius((f - 32) * 5 / 9) }


func CheckSomething() {
  var c Celsius
	var f Fahrenheit
  fmt.Println(c == 0)          // "true"
  fmt.Println(f >= 0)          // "true"
  fmt.Println(c == f)          // compile error: type mismatch
  fmt.Println(c == Celsius(f)) // "true"! 因为都是0值
}
```

## 包初始化

1. 我们可以用一个特殊的init初始化函数来简化初始化工作

2. 初始化工作是自下而上进行的，main包最后被初始化。

   - 以这种方式，可以确保在main函数执行之前，所有依赖的包都已经完成初始化工作了。

   - 如果一个p包导入了q包，那么在p包初始化的时候可以认为q包必然已经初始化过了。

3. 有些时候我们import一个包，但是实际上只需要他的init函数做一些事，并不需要使用。为了避免编译报错，我们可以采用匿名import的方式。比如说mysql driver的导入 `_ "github.com/go-sql-driver/mysql"`

## 变量作用域

当编译器遇到一个名字引用时，它会对其定义进行查找，查找过程从最内层的词法域向全局的作用域进行。如果查找失败，则报告“未声明的名字”这样的错误。如果该名字在内部和外部的块分别声明过，则内部块的声明首先被找到。在这种情况下，<font color=red>内部声明屏蔽了外部同名的声明</font>，让外部的声明的名字无法被访问。

## 字母大写的写法

```go

		x := "hello"
		// 1. 利用字符串+
    for _, x := range x {
        x := x + 'A' - 'a'
        fmt.Printf("%c", x) // "HELLO" (one letter per iteration)
    }
		// 2. 利用strings包
    fmt.Println(strings.ToUpper(x)) // "HELLO"
```

## 常见位运算符

```go

x := 3
// 1. 左移
fmt.Println(x << 2) // 3 * (2^2) = 3 * 4 = 12
// 2. 右移
fmt.Println(x >> 2) // 3 / (2^2) = 3 / 4 = 0
// 3. 异或
fmt.Println(x^3) // 任何数跟自身做异或运算，结果是0，
// 4. 按位置零。
// 如果对应y中bit位为1的话，表达式z = x &^ y结果z的对应的bit位为0
// 否则z对应的bit位等于x相应的bit位的值。
fmt.Println(8&^3) // 结果为8
```

## math.IsNaN

<font color=red>因为NaN和任何数比较都是不相等的</font>

```go
// NaN 一般用于表示无效的除法操作结果0/0或Sqrt(-1).
nan := math.NaN()
fmt.Println(nan == nan, nan < nan, nan > nan) // "false false false"
```

## 字符串赋值

<font color=red>字符串的值是不可变的</font>，但是我们也可以给一个字符串变量分配一个新字符串值。

```go
func main() {
  s := "left foot"
  t := s
  s += ", right foot"
  fmt.Println(s) // "left foot, right foot", 变量s将因为+=语句持有一个新的字符串值
	fmt.Println(t) // "left foot", t依然是包含原先的字符串值
  s[0] = 'L' // compile error: cannot assign to s[0]。因为尝试修改字符串内部数据的操作是被禁止的。
}
```

## itoa生成器

在一个const声明语句中，在第一个声明的常量所在的行，iota将会被置为0，然后在每一个有常量声明的行加一。

**练习 3.13：** 编写KB、MB的常量声明，然后扩展到YB。

```go
const (
	B float64 = 1 << (iota * 10)
	MB
  GB
  TB
  PB
  EB
  ZB
  YB
)
```



## Slice 内存复用

```go
package main

import "fmt"

// nonempty returns a slice holding only the non-empty strings.
// The underlying array is modified during the call.
func nonempty(strings []string) []string {
    i := 0
    /**
    	对于所有的 range 循环，Go 语言都会在编译期将原切片或者数组赋值给一个新的变量 `ha`，
    	在赋值的过程中就发生了拷贝，所以我们遍历的切片已经不是原始的切片变量了。
    */
    for _, s := range strings {
        if s != "" {
            strings[i] = s
            i++
        }
    }
    return strings[:i]
}
// nonempty函数也可以使用append函数实现
func nonempty2(strings []string) []string {
    out := strings[:0] // zero-length slice of original
    for _, s := range strings {
        if s != "" {
            out = append(out, s)
        }
    }
    return out
}
```

## JSON

```go
// json.MarshalIndent函数将产生整齐缩进的输出。
data, err := json.MarshalIndent(movies, "", "    ")
if err != nil {
    log.Fatalf("JSON marshaling failed: %s", err)
}
fmt.Printf("%s\n", data)
// Output: 
/**
	[
    {
        "Title": "Casablanca",
        "released": 1942,
        "Actors": [
            "Humphrey Bogart",
            "Ingrid Bergman"
        ]
    }
  ]
*/
```

## 模版格式化

[来自GO语言圣经][https://books.studygolang.com/gopl-zh/ch4/ch4-06.html]

## 值传递

实参通过值的方式传递，因此函数的形参是实参的拷贝。对形参进行修改不会影响实参。但是，如果实参包括引用类型，如指针，slice(切片)、map、function、channel等类型，实参可能会由于函数的间接引用被修改。

## 控制输出的缩进

利用fmt.Printf的一个小技巧控制输出的缩进。`%*s`中的`*`会在字符串之前填充一些空格。在例子中，每次输出会先填充`depth*2`数量的空格，再输出""，

```go
fmt.Printf("%*s</%s>\n", depth*2, "", n.Data)
```

## 匿名函数

通过匿名方式定义的函数可以访问完整的词法环境，这意味着在函数中定义的内部函数可以引用该函数的变量。

```go
// squares返回一个匿名函数。
// 该匿名函数每次被调用时都会返回下一个数的平方。
func squares() func() int {
    var x int
    return func() int {
        x++
        return x * x
    }
}
func main() {
    f := squares()
    fmt.Println(f()) // "1"
    fmt.Println(f()) // "4"
    fmt.Println(f()) // "9"
    fmt.Println(f()) // "16"
}
```



## 接口

一个类型如果拥有一个接口需要的所有方法，那么这个类型就实现了这个接口。

## Stringer重写

1. ### [Stringer](http://code.google.com/p/go/source/browse/src/pkg/fmt/print.go?name=release#63)

   - 重写了Stringer接口的类型（即有String方法），当采用任何接受字符的verb（%v %s %q %x %X）动作格式化一个操作数时，或者被不使用格式字符串如Print函数打印操作数时，会调用String方法来生成输出的文本。

2. ### [GoStringer](http://code.google.com/p/go/source/browse/src/pkg/fmt/print.go?name=release#71)

   - 重写了GoStringer接口的类型，当采用verb %#v格式化一个操作数时，会调用GoString方法来生成输出的文本。

```go
// Printf函数中%v参数包含的#副词，它表示用和Go语言类似的语法打印值。
// 对于结构体类型来说，将包含每个成员的名字。
fmt.Printf("%#v\n", w)
// Output:
// Wheel{Circle:Circle{Point:Point{X:42, Y:8}, Radius:5}, Spokes:20}
```



## 避免内存拷贝

**背景：** 因为Write方法需要传入一个byte切片，而我们希望写入的值是一个字符串。

```go
// writeString writes s to w.
// If w has a WriteString method, it is invoked instead of w.Write.
func writeString(w io.Writer, s string) (n int, err error) {
    type stringWriter interface {
        WriteString(string) (n int, err error)
    }
    // 利用类型断言，判断该类型是否有实现 stringWriter，有则使用，可避免内存拷贝
    // 之所以要用断言，是因为没法保证所有的io.Writer都实现了stringWriter
    if sw, ok := w.(stringWriter); ok {
        return sw.WriteString(s) // avoid a copy
    }
    // 直接用[]byte转化，用发生内存拷贝分配，可能对性能有影响 
    return w.Write([]byte(s)) // allocate temporary copy
}

func writeHeader(w io.Writer, contentType string) error {
    if _, err := writeString(w, "Content-Type: "); err != nil {
        return err
    }
    if _, err := writeString(w, contentType); err != nil {
        return err
    }
    // ...
}
```

## 接口的使用

1. **第一种是方法抽象**，实现类似JAVA多态的效果。一个接口的方法表达了实现这个接口的具体类型间的相似性，但是隐藏了代码的细节和这些具体类型本身的操作。重点在于方法上，而不是具体的类型上。**以io.Reader，io.Writer，fmt.Stringer，sort.Interface，http.Handler和error为典型。 **
2. **第二种是类型断言**，实现类似泛型的效果。类型断言用来动态地区别这些类型，使得对每一种情况可以有不同的处理方式。在这个方式中，重点在于具体的类型满足这个接口，而不在于接口的方法。

## sync.Mutex互斥锁

<font color=red>mutex不可重入。GO本身也没有重入锁。</font>

每次都会加锁、释放锁，所有需要这个方法的请求都是串行，处理效率会降低。

## sync.RWMutex读写锁

适用于“多读单写”场景。读操作可以并发，写操作会阻塞。

## sync.Once

惰性初始化，可用于实现单例模式。比如一个初始化常量数据的方法，为了避免并发的情况下，读取的时候无法确保是否已经初始化完毕，则可以使用此语法。

其底层实际也是用到了互斥锁，只是加多了一个是否需要初始化的标记，如果标记done为0，所以还没初始化，则上锁初始化。

## GOMAXPROCS

Go的调度器使用了一个叫做GOMAXPROCS的变量来决定会有多少个操作系统的线程同时执行Go的代码。

其默认的值是运行机器上的CPU的核心数，所以在一个有8个核心的机器上时，调度器一次会在8个OS线程上去调度GO代码。

## Go 闪电编译背后的真相

1. 所有导入的包必须在每个文件的开头显式声明，这样的话编译器就没有必要读取和分析整个源文件来判断包的依赖关系。
2. 禁止包的环状依赖，因为没有循环依赖，包的依赖关系形成一个有向无环图，每个包可以被独立编译，而且很可能是被并发编译。

## 包的匿名导入

**背景:**  这种方式通常是用来实现一个编译时机制，然后通过在main主程序入口选择性地导入附加的包。有时候我们只是想利用导入包而产生的副作用（计算包级变量的初始化表达式和执行导入包的init初始化函数）。但是如果只是导入一个包而并不使用导入的包将会导致一个编译错误。

**使用方式:** 可以用下划线`_`来重命名导入的包。但是，<font color=redea>下划线_为空白标识符，并不能被访问。</font>

**适用场景: **比如说mysql driver的导入，`import _ "github.com/go-sql-driver/mysql"`

## 编写有效的测试

1. 一个好的测试不应该引发其他无关的错误信息，它只要清晰简洁地描述问题的症状即可，有时候可能还需要一些上下文信息。
2. 一个好的测试不应该在遇到一点小错误时就立刻退出测试，它应该尝试报告更多的相关的错误信息，因为我们可能从多个失败测试的模式中发现错误产生的规律。

## 深度相等判断

```go
// 来自reflect包的DeepEqual函数可以对两个值进行深度相等判断。
// 它可以工作在任意的类型上，甚至对于一些不支持==操作运算符的类型也可以工作
func TestSplit(t *testing.T) {
    got := strings.Split("a:b:c", ":")
    want := []string{"a", "b", "c"};
    // true
  	fmt.Println(reflect.DeepEqual(got, want))
}
// 也存在不足，它将一个nil值的map和非nil值但是空的map视作不相等
// 同样nil值的slice 和非nil但是空的slice也视作不相等。
func TestCompare(t *testing.T) {
   var s1 []int
   s2 := make([]int,0)
    // false
   fmt.Println(reflect.DeepEqual(s1, s2))
}
```

