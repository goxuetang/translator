# Go高性能系列教程

原文链接 <https://dave.cheney.net/high-performance-go-workshop/gophercon-2019.html#benchmarking>

# 1. 基准测试

>三思而后行（Measure twice and cut once）

在我们试图改进程序性能之前，我们首先要知道程序的当前性能。
本节主要关注使用Go testing包如何构建有用的基准测试，并且给出一些最佳实践避免踩坑。

## 1.1 基准测试基本原则
在进行基准测试之前，你必须要有一个稳定的环境以得到可重复的输出结果。
+ 机器必须是空闲状态--不能在共享的硬件上采集数据，当长时间运行基准测试时不能浏览网页等
+ 机器是否关闭了节能模式。一般笔记本电脑上会默认开启该模式。
+ 避免使用虚拟机和云主机。一般情况下，为了尽可能地提高资源的利用率，虚拟机和云主机 CPU 和内存一般会超分配，超分机器的性能表现会非常地不稳定。

如果负担得起，请购买专用的性能测试硬件。 机架安装，禁用所有电源管理和热量缩放功能，并且永远不要在这些计算机上更新软件。 最后一点是从系统管理的角度来看糟糕的建议，但是如果软件更新改变了内核或库的执行方式-想想Spectre补丁-这将使以前的任何基准测试结果无效。

对于其他的原则，请进行前后采样，然后多次运行以获取一致的结果。

## 1.2  使用testing包构建基准测试
testing包中已经内置了基准测试的功能。如果我们有一个如下简单的函数：
```golang
func Fib3(n int) int {
	switch n {
	case 0:
		return 0
	case 1:
		return 1
	case 2:
		return 1
	default:
		return Fib(n-1) + Fib(n-2)
	}
}
```
我们可以通过testing包来写基准测试，基准测试的代码如下：
```golang
func BenchmarkFib20(b *testing.B) {
	for n := 0; n < b.N; n++ {
		Fib(20) //执行b.N次Fib函数
	}
}

func BenchmarkFib28(b *testing.B) {
	for n := 0; n < b.N; n++ {
		Fib(28) //执行b.N次Fib函数
	}
}
```
>注意：基准测试函数应该写在文件名后缀是 _test.go的文件中

基准测试类似单元测试，唯一的不同就是在测试函数中传的参数类型是 ___*testing.B___，而非 ___*testing.T___。这两个类型都实现了 ___testing.TB___接口，该接口提供了常用的Errorf(),Fatalf()和FailNow()常用函数。

### 1.2.1 执行一个包下的基准测试
因为基准测试使用的是testing包，所以要执行基准测试函数需要使用go test命令。但是，默认情况下，当我们调用go test的时候，基准测试会被排除在外，只执行单元测试。

所以，需要在go test命令中添加 ___-bench___标记，以执行基准测试。-bench标记使用一个正则表达式来匹配要运行的基准测试函数名称。所以，最常用的方式就是通过 *** -bench=.*** 标记来执行该包下的所有的基准函数。如下：
```golang
% go test -bench=. ./examples/fib/
goos: darwin
goarch: amd64
pkg: high-performance-go-workshop/examples/fib
BenchmarkFib20-8           28947             40617 ns/op
PASS
ok      high-performance-go-workshop/examples/fib       1.602s
```

> **go test**在匹配基准测试之前会执行所有的单元测试，如果你的代码里有很多单元测试，或者单元测试会耗费很长的时间，你可以通过go test的-run参数将单元测试排除掉。例如：
> ```golang
> % go test -run=none
> ```

### 1.2.2 基准测试工作原理
每个基准函数被执行时都有一个不同的b.N值，这个值代表基准函数应该执行的迭代次数。

b.N从1开始，如果基准函数在1秒内就执行完了，那么b.N的值会递增以便基准函数再重新执行（_译者注：即基准函数默认要运行1秒，如果该函数的执行时间在1秒内就运行完了，那么就递增b.N的值，重新再执行一次_）

b.N按照近似顺序增加，每次迭代大约增长20%。基准框架试图更智能，如果它看到较小的b.N值相对较快的完成了迭代，它将b.N增加的更快。

在上面的BenchmarkFi20-8的例子中，我们发现迭代大约29000次耗时超过了1秒。依据此数据，基准框架计算得知，平均每次运行耗时40617纳秒。

> BenchmarkFi20-8中的***-8***后缀是和运行该测试用例时的***GOMAXPROCS***值有关系。和***GOMAXPROCS***一样，此数字默认为启动时Go进程可见的CPU数
> 
> ```golang
> % go test -bench=. -cpu=1,2,4 ./examples/fib/
> goos: darwin
> goarch: amd64
> pkg:high-performance-go-workshop/examples/fib
> BenchmarkFib20             31479             37987 ns/op
> BenchmarkFib20-2           31846             37859 ns/op
> BenchmarkFib20-4           31716             39255 ns/op
> PASS
> ok      high-performance-go-workshop/examples/fib       4.805s
> ```
> 该示例展示了分别用CPU为1核、2核、4核时运行基准测试的结果。在该案例中，该参数对结果> > 几乎没有影响，因为该基准测试的代码是完全顺序执行的。

### 1.2.3 Go 1.13中基准测试框架的一些改变
在Go1.13版本之前，基准测试的迭代次数四舍五入为1、2、3、5的序列增长，这种四舍五入的初衷是使更便于肉眼阅读（make it easier to eyeball times)。然而，正确的分析都需要工具才能进行，因此，随着工具的改进，人们容易理解的数字变得不那么有价值。

四舍五入可能会隐藏一个数量级的变化。

幸运的是，在Go 1.13版本中，四舍五入的方式已经被移除了，这提高了在低的单位操作耗时（ns/op)的准确性，并随着基准测试框架更快的达到正确的迭代次数而减少了整体基准测试的运行时间。

### 1.2.4 改进基准测试的准确性
fib函数是一个特意设计的例子--除非我们正在为TechPower web项目写基准测试--计算斐波那契数列中第20个数字的素材也不太可能会影响您的业务。但是，该示例提供了一个编写基准测试的很好的示例。

具体来说，你希望你的基准测试可以运行数万次迭代，以便你可以获得一个较为准确的平均耗时。如果你的基准测试只执行了100次或10次迭代，那么最终得出的平均值可能会偏高。如果你的基准测试执行了上百万或十亿次迭代，那么得出的平均耗时将会非常准确，但这受代码布局的限制。

可以使用***-benchtime***标识增加基准测试执行的时间的方式来增加迭代的次数。例如：
```golang
% go test -bench=. -benchtime=10s ./examples/fib/
goos: darwin
goarch: amd64
pkg: high-performance-go-workshop/examples/fib
BenchmarkFib20-8          313048             41673 ns/op
PASS
ok      high-performance-go-workshop/examples/fib       13.442s
```
运行相同的基准测试，直到其达到b.N的值需要花费超过10秒的时间才能返回。由于我们的运行时间增加了10倍，因此迭代的总次数也增加了10倍。结果（每次操作耗时 41673ns/op)没有太大的变化，这就是我们所期望的。

***为什么总的耗时是13秒，而不是10秒呢？***
如果你又一个基准测试运行了数百万次或数十亿次迭代，导致每次操作的时间都在微秒或纳秒范围内，则你可能会发现基准值不稳定，因为你的机器硬件的散热性能、内存局部性、后台进程、gc等因素。

对于每次操作在10纳秒以下的，指令重新排序，并且代码对齐的相对论效应将影响基准时间。

通过-count标志，可以指定基准测试多跑几次：
```golang
% go test -bench=Fib20 -count=10 ./examples/fib/ | tee old.txt
goos: darwin
goarch: amd64
pkg: high-performance-go-workshop/examples/fib
BenchmarkFib20-8           30099             38117 ns/op
BenchmarkFib20-8           31806             40433 ns/op
BenchmarkFib20-8           30052             43412 ns/op
BenchmarkFib20-8           28392             39225 ns/op
BenchmarkFib20-8           28270             42956 ns/op
BenchmarkFib20-8           28276             49493 ns/op
BenchmarkFib20-8           26047             45571 ns/op
BenchmarkFib20-8           27392             43803 ns/op
BenchmarkFib20-8           27507             44896 ns/op
BenchmarkFib20-8           25647             43579 ns/op
PASS
ok      high-performance-go-workshop/examples/fib       16.516s
```

## 1.3 使用benchstat工具比较基准测试
在上面我建议运行多次基准测试以便获取更多的数据来求平均值。对于任何一个基准测试来说，这是一个非常好的建议，由于基准测试受电源管理、后台进程、散热的影响。

接下来，我将介绍一个由Russ Cox编写的工具：**benchstat**
```golang
% go get golang.org/x/perf/cmd/benchstat
```

Benchstat可以进行一组基准测试，并告诉你他们的稳定性。这是Fib(20)函数在使用电池的电脑上执行的基准示例：
```golang
% go test -bench=Fib20 -count=10 ./examples/fib/ | tee old.txt
goos: darwin
goarch: amd64
pkg: high-performance-go-workshop/examples/fib
BenchmarkFib20-8           30721             37893 ns/op
BenchmarkFib20-8           31468             38695 ns/op
BenchmarkFib20-8           31726             37521 ns/op
BenchmarkFib20-8           31686             37583 ns/op
BenchmarkFib20-8           31719             38087 ns/op
BenchmarkFib20-8           31802             37703 ns/op
BenchmarkFib20-8           31754             37471 ns/op
BenchmarkFib20-8           31800             37570 ns/op
BenchmarkFib20-8           31824             37644 ns/op
BenchmarkFib20-8           31165             38354 ns/op
PASS
ok      high-performance-go-workshop/examples/fib       15.808s

% benchstat old.txt
name     time/op
Fib20-8  37.9µs ± 2%
```
benchstat告诉我们，Fib20-8的平均操作耗时是38.8微妙，并且误差在+/-2%。这是因为在运行基准测试期间，我没动过机器。

### 1.3.1 改进Fib函数
确定两组基准测试之间的性能差异可能是非常乏味且容易出错的。Benchstat工具可以帮助我们做这个事情。
> 保存基准测试的输出结果是非常有用的，同时，你也需要保存产生它的二进制文件。这个会让你有机会重新执行之前的基准测试。为了达到这个目标，在执行go test时需要添加 -c标记以保存测试的二进制文件--同时我还经常将生成的二进制文件.text重命名为.golden
> ```golang
> % go test -c
> mv fib.test fib.golden
> ```

先前的Fib函数具有斐波那契数列中第0和第1个数字的硬编码值。在之后使用递归调用了自身。稍后，我们将讨论递归的成本，但目前，我们假设递归是有成本的，尤其是因为我们的算法使用的是指数时间。

对此的简单解决方法是对斐波那契数列中的另一个数字进行硬编码，从而将每个可回溯调用的深度减少一个。

```golang
func Fib(n int) int {
	switch n {
	case 0:
		return 0
	case 1:
		return 1
	case 2:
		return 1
	default:
		return Fib(n-1) + Fib(n-2)
	}
}
```

>该文件还包含针对Fib的全面测试。 如果没有通过验证当前行为的测试，请勿尝试提高基准。

为了能和我们的新版本进行比较，我们编译一个新的测试的二进制文件并对其进行了基准测试，并使用Benchstat工具比较输出。

```golang
% go test -c
% ./fib.golden -test.bench=. -test.count=10 > old.txt
% ./fib.test -test.bench=. -test.count=10 > new.txt
% benchstat old.txt new.txt
name     old time/op  new time/op  delta
Fib20-8  37.9µs ± 2%  24.1µs ± 3%  -36.26%  (p=0.000 n=10+10)
```
运行完上面的比较结果后，有2件事情需要确认：
+ 两次运行基准间上下浮动的值。1-2%是较好的，3-5%还可以，高于5%时就需要考虑你程序的稳定性了。要当心当差异较大时，请不要贸然改进性能。
+ 样本缺失。benchstat工具将报告有多少有效的样本数据。有时即使你执行了10次，但也可能只发现了9个样本。10%或更低的拒绝率是可以接受的，高于10%可能表明您的设置不稳定，并且你可能比较的样本太少。

### 1.3.2 注意p值
低于0.05的p值可能具有统计学意义。 p值大于0.05表示基准可能没有统计意义。

## 1.4 避免基准测试的启动耗时
有时候你的基准测试每次执行的时候会有一次启动配置耗时。b.ResetTimer()函数可以用于忽略启动的累积耗时。
```golang
func BenchmarkExpensive(b *testing.B) {
	boringAndExpensiveSetup()
	b.ResetTimer()
	for n := 0; n < b.N; n++ {
		//function under test
	}
}
```
在上例代码中，使用b.ResetTimer()函数重置了基准测试的计时器

如果在每次循环迭代中，你有一些费时的配置逻辑，要使用b.StopTimer()和b.StartTimer()函数来暂定基准测试计时器。
```golang
func BenchmarkComplicated(b *testing.B) {
	for n := 0; n < b.N;n++ {
		b.StopTimer()
		complicatedSetup()
		b.StartTimer()
		//function under test
	}
}
```
+ 上例中，先使用b.StopTimer()暂停计时器
+ 然后执行完复杂的配置逻辑后，再使用b.StartTimer()启动计时器

通过以上两个函数，则可以忽略掉启动配置所耗费的时间。

## 1.5 基准测试的内存分配
内存分配的次数和分配的大小和基准测试的执行时间强相关。你可以通过在代码中增加b.ReportAllocs()函数来告诉testing框架记录内存分配的数据。
```golang
func BenchmarkRead(b *testing.B) {
	b.ReportAllocs()
	for n := 0; n < b.N; n++ {
		//function under test
	}
}
```
下面是使用bufio包中的基准测试的一个示例：
```golang
% go test -run=^$ -bench=. bufio
goos: darwin
goarch: amd64
pkg: bufio
BenchmarkReaderCopyOptimal-8            12999212                78.6 ns/op
BenchmarkReaderCopyUnoptimal-8           8495018               133 ns/op
BenchmarkReaderCopyNoWriteTo-8            360471              2805 ns/op
BenchmarkReaderWriteToOptimal-8          3839959               291 ns/op
BenchmarkWriterCopyOptimal-8            13878241                82.7 ns/op
BenchmarkWriterCopyUnoptimal-8           9932562               117 ns/op
BenchmarkWriterCopyNoReadFrom-8           385789              2681 ns/op
BenchmarkReaderEmpty-8                   1863018               640 ns/op            4224 B/op          3 allocs/op
BenchmarkWriterEmpty-8                   2040326               579 ns/op            4096 B/op          1 allocs/op
BenchmarkWriterFlush-8                  88363759                12.7 ns/op             0 B/op          0 allocs/op
PASS
ok      bufio   13.249s
```

>你也可以使用 go test -benchmem标识来强制testing框架打印出所有基准测试的内存分配次数
>```golang
>%  go test -run=^$ -bench=. -benchmem bufio
goos: darwin
goarch: amd64
pkg: bufio
BenchmarkReaderCopyOptimal-8            13860543                82.8 ns/op            16 B/op          1 allocs/op
BenchmarkReaderCopyUnoptimal-8           8511162               137 ns/op              32 B/op          2 allocs/op
BenchmarkReaderCopyNoWriteTo-8            379041              2850 ns/op           32800 B/op          3 allocs/op
BenchmarkReaderWriteToOptimal-8          4013404               280 ns/op              16 B/op          1 allocs/op
BenchmarkWriterCopyOptimal-8            14132904                82.7 ns/op            16 B/op          1 allocs/op
BenchmarkWriterCopyUnoptimal-8          10487898               113 ns/op              32 B/op          2 allocs/op
BenchmarkWriterCopyNoReadFrom-8           362676              2816 ns/op           32800 B/op          3 allocs/op
BenchmarkReaderEmpty-8                   1857391               639 ns/op            4224 B/op          3 allocs/op
BenchmarkWriterEmpty-8                   2041264               577 ns/op            4096 B/op          1 allocs/op
BenchmarkWriterFlush-8                  87643513                12.5 ns/op             0 B/op          0 allocs/op
PASS
ok      bufio   13.430s
>```

## 1.6 注意编译器的优化
下面的示例来源于[issue 14813](https://github.com/golang/go/issues/14813#issue-140603392)
```golang
const m1 = 0x5555555555555555
const m2 = 0x3333333333333333
const m4 = 0x0f0f0f0f0f0f0f0f
const h01 = 0x0101010101010101

func popcnt(x uint64) uint64 {
	x -= (x >> 1) & m1
	x = (x & m2) + ((x >> 2) & m2)
	x = (x + (x >> 4)) & m4
	return (x * h01) >> 56
}

func BenchmarkPopcnt(b *testing.B) {
	for i := 0; i < b.N; i++ {
		popcnt(uint64(i))
	}
}
```
你认为这个基准测试的性能到底有多快呢？让我看下面结果
```golang
% go test -bench=. ./examples/popcnt/
goos: darwin
goarch: amd64
pkg: high-performance-go-workshop/examples/popcnt
BenchmarkPopcnt-8       1000000000               0.278 ns/op
PASS
ok      high-performance-go-workshop/examples/popcnt    0.318s
```
0.278纳秒；基本上就是一个cpu时钟的时间。即使假设每个时钟周期中CPU有一些指令要运行，但这个数字看起来也有点不太合理。那到底发生了什么？

想要了解到底发生了什么，我们需要看下基准测试下的popcnt函数。popcnt函数是一个叶子函数-即该函数没有调用其他任何函数-所以编译器可以内联它。

因为该函数是内联函数，编译器可以知道该函数没有任何副作用。popcnt函数不会影响任何全局变量的状态。因此，调用被消除。下面是编译器看到的：
```golang
func BenchmarkPopcnt(b *testing.B) {
	for i := 0; i < b.N; i++ {
		//optimised away
	}
}
```
在我测试过的所有版本的Go编译器上，仍然会生成循环。 但是英特尔CPU确实擅长优化循环，尤其是空循环。

### 1.6.1 练习，看汇编
在我们继续之前，让我们看下汇编以确定我们看到的
```golang
% go test -gcflags=-S
```
+ 说明：使用`gcflags="-l-S"`标识可以禁用内联，那对汇编的输出有什么影响

> ***优化是一件好的事情***
> 值得注意的是，通过消除不必要的计算，使实际代码快速运行的优化与消除没有明显副作用的基准测试的优化是一样的。
> 随着Go编译器的改进，这种情况会越来越普遍
### 1.6.2 修复基准测试
禁用内联以使基准测试可以正常工作是不现实的。我们想在编译器优化的基础上编译我们的代码。

为了修复这个基准测试，我们必须确保编译器不能证明BenchmarkPopcnt的主体不会导致全局状态改变。（***译者注：即让编译器知道BenchmarkPopcnt函数有可能会改变全局状态，这样编译器就不用再将函数做内联优化了***）

```golang
var Result uint64

func BenchmarkPopcnt(b *testing.B) {
	var r uint64
	for i := 0; i < b.N; i++ {
		r = popcnt(uint64(i))
	}
	Result = r
}
```
以上通过增加全局变量Result是比较推荐的方式，以此来确保编译器不会对循环主题进行优化。

首先，我们把popcnt函数的调用结果存储在变量r中。其次，因为r是局部变量，一旦基准测试结束，变量r的生命周期也将结束，所以最后我们把r的结果赋值给全局变量Result。

因为变量Result是全局可见，所以编译器不能确定其他导入该包的代码是否也在使用该变量，因此编译器不能对该赋值操作进行优化。

## 1.7 基准测试错误
在基准测试中，for循环是至关重要的。

这里是两个错误的基准测试，你能解释他们为什么错误吗？

```golang
func BenchmarkFibWrong(b *testing.B) {
	Fib(b.N)
}

func BenchmarkFibWrong2(b *testing.B) {
	for n := 0; n < b.N; n++ {
		Fib(n)
	}
}
````
运行上面的基准测试试一试，你将看到什么？

## 1.8 基准测试中使用math/rand
众所周知，计算机非常擅长预测并缓存（译者注：即cpu的局部性原理）。也许我们的Popcnt基准测试返回的是一个缓存的结果。让我们看一下下面的例子：
```golang
var Result uint64

func BenchmarkPopcnt(b *testing.B) {
	var r uint64
	for i := 0; i < b.N; i++ {
		r = popcnt(rand.Uint64())
	}
	
	Result = r
}
```
以上代码是可靠的吗？如果不是，哪里出错了？

## 1.9 收集基准测试数据
该testing包内置了对生成CPU，内存和模块配置文件的支持。
+ **-cpuprofile=$FILE** 收集CPU性能分析到$FILE文件
+ **-memprofile=$FILE**,将内存性能分析写入到$FILE文件，-memprofilerate=N 调节采样频率为1/N
+ -blockprofile=$FILE,输出内部goroutine阻塞的性能分析文件数据到$FILE

这些标识也同样可以用于二进制文件
```golang
% go test -run=XXX -bench=. -cpuprofile=c.p bytes
% go tool pprof c.p
```

### benchmark小结
benchmark是go语言中用于测试性能的一个工具。主要适用于在已知性能瓶颈在哪里时的场景。该测试函数位于_test.go为结尾的文件中，性能测试函数名以Benchmark开头，可以测试出被执行函数被执行的次数，平均每次执行所消耗的时间，以及cpu以及内存的性能数据。 同时，在执行基准测试时也需要注意运行环境的稳定性，执行的次数，求得的平均值越准确。

