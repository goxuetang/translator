# Go高性能系列教程之五：内存和垃圾回收

原文链接 <https://dave.cheney.net/high-performance-go-workshop/gophercon-2019.html#memory-and-gc>

# 5. 内存和垃圾回收

Go是一门自动垃圾收回的语言。这是设计原则，不能改变。

作为自动垃圾回收的语言，Go程序的性能通常取决于他们与垃圾收集器的交互。

除了算法的选择以外，内存的消耗是决定应用程序性能和可伸缩性的最重要的因素。

本节将讨论垃圾回收器的操作，如何评估程序的内存使用情况以及在垃圾回收器的性能成为瓶颈时如何降低内存使用量的策略。

## 5.1 垃圾回收器的目的

任何垃圾回收器的目的都是为了**让程序有足够的可用内存**。

你可能不太同意这个观点，但是这是垃圾回收器设计者工作时最基本的假设。

就总运行时间而言，STW、标记清除GC是最有效的垃圾回收算法。适用于批处理、模拟等。但是，随着时间的流式，Go GC从纯粹的STW（stop the world）转变成并发、非压缩的方式。这是因为Go GC被设计成了低延迟服务和交互程序。

Go GC的设计偏向于低延迟，而不再是高吞吐；它将一些内存分配成本转移到了mutator上，以减少后续的清理成本。

## 5.2 垃圾回收器的设计

在过去的几年里，Go GC的设计发生了很多改变：
+ Go 1.0，高度依赖于**tcmalloc**的STW、标记清除收集器
+ Go 1.3，非常精确的收集器（fully precise collector），不会将堆内存上的大数字误认为是指针了，因此**降低了内存浪费**。
+  Go 1.5，一个新的GC设计，主要关注在**低延迟**而非高吞吐量上。
+  Go 1.6，改进GC，用低延迟处理大的堆内存。
+  Go 1.7，一些小的改进，主要是重构。
+  Go 1.8，更进一步**降低STW的时间**，目前降低到了100微秒以内。
+  Go 1.10+， 摆脱了纯粹的goroutine协同调度以**降低**触发整个GC周期的**延迟**。
+  Go 1.13，重写Scavenger

### 5.2.1 垃圾回收调整
Go运行时提供了一个环境变量来调整GC，***GOGC***
GOGC的公式是：
```shell
goal = reachable * （1 + GOGC/100）
```
该公式中，reachable是当前的内存量，goal是GC运行的目标堆内存量，即当堆内存量达到该值时就需要执行GC了。

>GOGC是Go运行时很早就支持的一个环境变量。可能比GOROOT的支持还早。GOGC的值能够影响GC执行的频率。默认值是100.即当分配的堆内存是目前的一倍的时候，则会运行GC。

例如，如果我们现在有一个256M大小的堆内存，同时，GOGC=100（默认），当堆内存增加到以下值时，GC将会执行：
```shell
512MB = 256MB * (1 + 100 / 100)
```
+ GOGC变量值大于100时会导致堆内存增长过快，这样会减少GC的压力。
+ GOGC小于100时，导致堆堆内存较慢的增长，会增大GC的压力。

GOGC的默认值100只是一个指导值。你可以根据你线上应用的实际负载情况选择合适的值。

### 5.2.2 VSS和scavenger
很多应用程序都会有不同的阶段。启动阶段、稳定运行阶段和（可选）结束阶段。每个阶段都有不同的内存分析数据。启动阶段可能会处理或汇总大量的数据。稳定运行阶段可能会消耗和客户端连接数或请求数成比例的内存。关闭阶段可能会消耗和稳定运行阶段处理的数据量成正比的内存，以将数据汇总或写入到磁盘上。

实际上，您的应用程序在启动时可能会使用比其余阶段更多的内存，然后它的堆将超过必需的内存，但大部分未使用。 如果Go运行时可以告诉操作系统哪部分堆内存是没有被用到的，这将很有用。

> Go 1.13中的新特性
> 自从Go 1.1中实现了scavenger后，基本没有变更过。在Go 1.13中，scavenging从后台周期性的操作迁移到了某些命令驱动，因此，没有从scavenging受益的进程不会为此付出代价，因为内存分配变化很大且长时间运行的程序应该会更有效的将内存归还给操作系统。
> 但是，一些与清除相关的 CL 尚未提交。 这项工作可能要到 Go 1.14 才能完成。


### 5.2.3 GC监控

一种监控垃圾收集器最简单的方法是启用GC日志记录输出。

这些统计信息始终会收集，但通常会被禁止显示，您可以通过设置***GODEBUG***环境变量来启用它们的显示。

```shell
% env GODEBUG=gctrace=1 godoc -http=:8080
gc 1 @0.012s 2%: 0.026+0.39+0.10 ms clock, 0.21+0.88/0.52/0+0.84 ms cpu, 4->4->0 MB, 5 MB goal, 8 P
gc 2 @0.016s 3%: 0.038+0.41+0.042 ms clock, 0.30+1.2/0.59/0+0.33 ms cpu, 4->4->1 MB, 5 MB goal, 8 P
gc 3 @0.020s 4%: 0.054+0.56+0.054 ms clock, 0.43+1.0/0.59/0+0.43 ms cpu, 4->4->1 MB, 5 MB goal, 8 P
gc 4 @0.025s 4%: 0.043+0.52+0.058 ms clock, 0.34+1.3/0.64/0+0.46 ms cpu, 4->4->1 MB, 5 MB goal, 8 P
gc 5 @0.029s 5%: 0.058+0.64+0.053 ms clock, 0.46+1.3/0.89/0+0.42 ms cpu, 4->4->1 MB, 5 MB goal, 8 P
gc 6 @0.034s 5%: 0.062+0.42+0.050 ms clock, 0.50+1.2/0.63/0+0.40 ms cpu, 4->4->1 MB, 5 MB goal, 8 P
gc 7 @0.038s 6%: 0.057+0.47+0.046 ms clock, 0.46+1.2/0.67/0+0.37 ms cpu, 4->4->1 MB, 5 MB goal, 8 P
gc 8 @0.041s 6%: 0.049+0.42+0.057 ms clock, 0.39+1.1/0.57/0+0.46 ms cpu, 4->4->1 MB, 5 MB goal, 8 P
gc 9 @0.045s 6%: 0.047+0.38+0.042 ms clock, 0.37+0.94/0.61/0+0.33 ms cpu, 4->4->1 MB, 5 MB goal, 8 P
```

跟踪结果显示了GC活动的通用测量结果。gctrace=1的输出格式是在[runtime包文档](https://golang.org/pkg/runtime/#hdr-Environment_Variables)中描述的。

DEMO: Show godoc with GODEBUG=gctrace=1 enabled
示例：打开GODEBUG=gctrace=1以展示godoc结果
>说明：在生产环境中使用该选项，对性能没有任何影响，

当你知道有问题的时候使用GODEBUG=gctrace=1是非常好的选择，但对于应用程序的常规检测，我推荐使用net/http/pprof接口。

```golang
import _ "net/http/pprof"
```

导入 net/http/pprof包的时候，将会在/debug/pprof中注册一个使用各种运行时指标的的处理器。包括：
+ 运行时协程列表，/debug/pprof/goroutine?debug=1
+ 静态内存分配报告，/debug/pprof/heap?debug=1

> 注意 net/http/pprof将使用默认的http.ServerMux注册自己。如果你使用http.ListenAndServe(address, nil)时，要当心会暴露自己。
> 
示例：godoc  -http=:8080, 展示/debug/pprof

## 5.3 最小化内存分配

事实上，内存分配都是有代价开销的，无论你的语言是否是自动垃圾回收的还是手动回收的。

内存分配可能是整个代码库的开销。每一个都只占运行时的一小部分时间，但总的来说，他们代表了相当大的成本。因为这个成本贯穿在很多地方，确定最大的开销者可能很复杂，并且通常需要重新设计API接口。

每次分配都应按需分配 。 打个比方：如果您因为打算建立家庭而搬到更大的房子，那将是对您的资本的充分利用。 如果您因为某人要您照顾孩子一个下午而搬到更大的房子，那将浪费您的资金。

### 5.3.1 strings vs []bytes

In Go string values are immutable, []byte are mutable.
在Go语言中，字符串值是不可变的，[]byte是可变的。

大多数应用程序都喜欢使用string，但是大多数IO操作都是用[]byte完成的。

尽可能的避免将[] byte转换为字符串，这通常意味着选择一种表示形式，是选择字符串，还是选择[] byte来保存值。 如果您从网络或磁盘读取数据，则通常为[] byte。

Go中的bytes package 包含许多与string package相同的操作 -- Split，Compare，HasPrefix，Trim等。

在底层实现中，字符串和bytes使用相同的汇编原语。

### 5.3.2 用 []byte作为map的key
使用string作为map的key是非常常见的，但有时候你会使用字节数组（[]byte）作为map的key。换句话说，你可能具有[] byte形式的键，但是切片又没有定义等价的运算符，因此[]bytes不能用作map的key。

编译器针对这种情况实现了特定的优化：
```golang
var m map[string]string
v, ok := m[string(bytes)]
```

在map的key查找中，这将避免从byte slice到字符串的转换。这是非常特殊的，如果你做如下操作，那么将不会工作：
```golang
key := string(bytes)
val, ok := m[key]
```

### 5.3.3 []byte到string的转换
> 这是Go 1.13版本中的新特性

就像在map的key中会自动将[]byte转换成字符串一样，在比较两个[]byte切片是否相等时也需要将字节切片[]byte 转换成字符串后再进行比较-本质上是对字节切片的内容做了一个拷贝，或者是使用字节切片类型的比较函数：bytes.Equal

好消息是，在Go 1.13中编译器已经对字节切片转换成字符串进行了改进，下面的是对字节切片进行比较时避免了内存分配的测试：
```golang
func BenchmarkBytesEqualInline(b *testing.B) {
	x := bytes.Repeat([]byte{'a'}, 1<<20)
	y := bytes.Repeat([]byte{'a'}, 1<<20)
	b.ReportAllocs()
	b.ResetTimer()
	for n := 0; n < b.N; n++ {
		if string(x) != string(y) {
			b.Fatal("x != y")
		}
	}
}

func BenchmarkBytesEqualExplicit(b *testing.B) {
	x := bytes.Repeat([]byte{'a'}, 1<<20)
	y := bytes.Repeat([]byte{'a'}, 1<<20)
	b.ReportAllocs()
	b.ResetTimer()
	for n := 0; n < b.N; n++ {
		q := string(x)
		r := string(y)
		if q != r {
			b.Fatal("x != y")
		}
	}
}
```

更进一步阅读[https://go-review.googlesource.com/c/go/+/173323]

### 5.3.4 避免字符串连接
**Go语言中的字符串是不可变的，连接两个字符串将会产生第三个字符串**。下面的程序哪个会更快？
```golang
s := request.ID
s += " " + client.Addr().String()
s += " " + time.Now().String()
r = s
```

```golang
var b bytes.Buffer
fmt.Fprintf(&b, "%s %v %v", request.ID, client.Addr(), time.Now())
r = b.String()
```

```golang
r = fmt.Sprintf("%s %v %v", request.ID, client.Addr(), time.Now())
```

```golang
b := make([]byte, 0, 40)
b = append(b, request.ID...)
b = append(b, ' ')
b = append(b, client.Addr().String()...)
b = append(b, ' ')
b = time.Now().AppendFormat(b, "2006-01-02 15:04:05.999999999 -0700 MST")
r = string(b)
```

```golang
var b strings.Builder
b.WriteString(request.ID)
b.WriteString(" ")
b.WriteString(client.Addr().String())
b.WriteString(" ")
b.WriteString(time.Now().String())
r = b.String()
```

DEMO: go test -bench=. ./examples/concat

### 5.3.5 不要在你的API调用方上强制分配内存

确保你的API调用方减少内存垃圾生成的数量

考虑以下两种Read方法：
```golang
func (r *Reader) Read() ([]byte, error)
func (r *Reader) Read(buf []byte) (int, error)
```

第一个方法是不带任何参数的，并且返回一个[]byte。第二个函数时带一个[]byte的buf参数，并且返回的是读取到的字节数量。

第一个Read方法调用方总是会分配一个字节切片来接收返回的值，这在GC上带来压力。第二个方法是将读到的数据填充到已经给的buffer中。
***译者注：第一个方法是因为在Read函数中分配的字节切片指向的内存逃逸到了堆上，而堆内存是需要进行GC回收的，所以会增加GC的压力。但第二个方法是只有调用者分配了一次内存，被调用者共享了传入进来的内存***

### 5.3.6 如果切片长度可知则可预先分配

Append函数非常方便，但也比较浪费资源，

Slice的空间扩容首先会按成倍的方式直至扩容到1024个元素，然后每次扩容会按原容量25%的增速扩容。那么，以下代码中，我们使用append往b中添加1个或多个元素后其容量是多少？
 ```golang
 func main() {
 	b := make([]int, 1024)
 	b = append(b, 99)
 	fmt.Println("len:", len(b), "cap:", cap(b))
 }
 ```
如果你使用append模式，那么你可能会拷贝大量的数据，同时也会产生大量的垃圾。

**如果能提前知道切片的长度，然后预分配目标大小的切片容量，就可以避免数据的拷贝，并能依然能够确保容量的正确性**：

之前：
```golang
var s []string
for _, v := range fn() {
        s = append(s, v)
}
return s
```

之后：
```golang
vals := fn()
s := make([]string, len(vals))
for i, v := range vals {
        s[i] = v
}
return s
```

## 5.4 使用sync.Pool
> 注意：sync.Pool不是缓存。它在任何时刻都可能被清空。不要把任何重要的数据放在sync.Pool中，他们有可能会被删除。

sync包中的sync.Pool被用于对象的重用。
sync.Pool没有固定的大小或最大容量。你可以增加一个对象到sync.Pool中，也可以从sync.Pool中获取一个对象，直到有GC发生，sync.Pool将会被无条件的被清空。这是就这样[设计](https://groups.google.com/forum/#!searchin/golang-dev/gc-aware/golang-dev/kJ_R6vYVYHU/LjoGriFTYxMJ)的。

sync.Pool实践：
```golang
var pool = sync.Pool{New: func() interface{} { return make([]byte, 4096) }}

func fn() {
	buf := pool.Get().([]byte) // takes from pool or calls New
	// do work
	pool.Put(buf) // returns buf to the pool
}
```


## 5.5 重排结构体的字段顺序以便更好的压缩

考虑以下结构体的定义
```golang
type S struct {
	a bool
	b float64
	c int32
}
```
那么，一个该类型的变量值将占用多少内存呢？
```golang
var s S
fmt.Println(unsafe.Sizeof(s)) 
```
结果是在64位系统上是24字节，在32位系统上是16字节。

这是为什么呢？这不得不从**内存对齐说起（padding and alignment)**

float64类型的值在所有的平台上都是8字节的，因此它们必须始终位于8的倍数的地址处。这叫做自然对齐。因为CPU自然希望长度为4字节的字段在4字节边界上对齐，8字节的字段在8字节边界上对齐，依此类推。这是因为某些平台（尤其不是Intel）不允许您对未正确对齐的值进行操作。 即使在确实支持所谓的不对齐访问的平台上，访问这些字段通常也会产生成本。

了解了内存对齐之后，我们就能知道编译器是如何将这些字段排列在内存上的了：
```golang
type S struct {
	a bool
	_ [7]byte // padding 注释1
	b float64
	c int32
	_ [4]byte // padding 
}
```
+ 在a bool字段上，bool值原本占1个字节，所以需要额外填充7个字节，以确保b float64是从8字节的边界上开始的。
+ 在c int32字段删个，int32值原本占4个字节，所以需要额外填充4个字节以确保S数组或切片在内存上整体排列的正确性。

深入阅读:[Padding is hard](https://dave.cheney.net/2015/10/09/padding-is-hard)









