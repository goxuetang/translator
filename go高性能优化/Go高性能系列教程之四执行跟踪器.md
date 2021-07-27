# Go高性能系列教程之四：执行跟踪器

原文链接 <https://dave.cheney.net/high-performance-go-workshop/gophercon-2019.html#benchmarking>

# 4. 执行跟踪（tracer）

执行跟踪器（execution tracer）是由Dmity Vyukov为Go 1.5版本开发的，并且几年以来一直在文档记录和使用中。

不像采样分析检测，执行跟踪器是被集成到Go运行时环境的，所以，它能够知道Go程序在特定的时刻正在做什么。但是原理是什么呢？

## 4.1 什么是执行跟踪器，我们又为什么需要它？

我认为要解释什么是执行跟踪器，它又为什么如此重要。最简单的方式就是通过使用pprof的代码片段，用go tool pprof执行一段性能表现不佳的代码，看看有哪些方面是该工具覆盖不到的。

我们看下这段代码，代码地址位于[Francesc Compoy's mandelbrot package](https://github.com/campoy/mandelbrot)

```golang
cd examples/mandelbrot
go build && ./mandelbrot
```

如果我们编译并运行该程序，将产生如下的图片：


### 4.1.1 这个程序运行了多长时间

那么，这段程序生成一个1024*1024像素图片化了多长时间？

我知道的最简单的方式就是使用使用 time命令：

```golang
% time ./mandelbrot
real    0m1.654s
user    0m1.630s
sys     0m0.015s
```

> 注意：不要使用 time go run mandebrot.go,否则你将要包含编译该程序的时间。

### 4.1.2 这个程序做了什么？

那么，这个程序花了1.6秒的时间生成了一张曼德勃罗图片并输出了它。

这是比较好的吗？我们还能让它运行的再快点吗？
回答这个问题的一个方式就是使用Go内建的pprof工具来监控分析程序。

让我们试一试。

## 4.2 生成监控分析

为了生成监控分析文件我们有以下两种方式：

1. 直接用 runtime/pprof包
2. 用第三方包，像 github.com/pkg/profile

## 4.3 用 runtime/pprof包生成监控文件

为了表现方便，我们直接将cpuprofile的输出到os.Stdout中

```golang
import "runtime/pprof"

func main() {
	pprof.StartCPUProfile(os.Stdout)	
	defer pprof.StopCPUProfile()
}
```

通过增加函数开头的这部分代码，该程序运行时将把cpu的监测数据写入到os.Stdout中。

```golang
ccd examples/mandelbrot-runtime-pprof
go run mandelbrot.go > cpu.pprof
```

> 注意：我们可以直接使用 go run命令运行，因为cpu的监控分析文件只包含 mandelbrot.go的执行时间，并不会包含编译时间。

### 4.3.1 使用github.com/pkg/profile包 生成监控文件

上面我们展示了使用一种最简便的方式来生成监控分析文件，但存在一些问题：

+ 如果你在编写代码时忘记了将输出写入特定的文件中，你将会将终端会话卡死。
+ 如果你将其他内容也写入os.Stdout，例如，fmt.Println，那么，你的终端将会输出混合的内容。

所以，我们推荐的使用runtime/pprof的方式是将**跟踪内容写入到一个特定的文件中。并且在程序终止之前，你必须确保跟踪文件停止，以及文件被关闭**。

所以，前些年，我写了这个包来避免这些问题：

```golang
import "github.com/pkg/profile"func 

main() {
	defer profile.Start(profile.CPUProfile, profile.ProfilePath(".")).Stop()
}
```

如果我们运行该程序，我们会在当前目录下生成一个监控分析文件：

```shell
% go run mandelbrot.go
2017/09/17 12:22:06 profile: cpu profiling enabled, cpu.pprof
2017/09/17 12:22:08 profile: cpu profiling disabled, cpu.pprof
```

### 4.3.2 分析监控文件

现在我们已经得到了监控文件，我们可以使用go tool pprof工具来分析了。

```shelll
% go tool pprof -http=:8080 cpu.pprof
```

执行后，我们看到整个程序运行了1.81秒（***和之前的1.6秒相比增加了一些固定的profiling开销***）。我们也看到pprof采集数据花费了1.53秒，因为pprof是基于数据采样的，依赖于操作系统的SIGPROF计时器。

我们可以用 pprof中的top命令来对跟踪信息进行排序：

```shell
% go tool pprof cpu.pprof
Type: cpu
Time: Mar 24, 2019 at 5:18pm (CET)
Duration: 2.16s, Total samples = 1.91s (88.51%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 1.90s, 99.48% of 1.91s total
Showing top 10 nodes out of 35
      flat  flat%   sum%        cum   cum%
     0.82s 42.93% 42.93%      1.63s 85.34%  main.fillPixel
     0.81s 42.41% 85.34%      0.81s 42.41%  main.paint
     0.11s  5.76% 91.10%      0.12s  6.28%  runtime.mallocgc
     0.04s  2.09% 93.19%      0.04s  2.09%  runtime.memmove
     0.04s  2.09% 95.29%      0.04s  2.09%  runtime.nanotime
     0.03s  1.57% 96.86%      0.03s  1.57%  runtime.pthread_cond_signal
     0.02s  1.05% 97.91%      0.04s  2.09%  compress/flate.(*compressor).deflate
     0.01s  0.52% 98.43%      0.01s  0.52%  compress/flate.(*compressor).findMatch
     0.01s  0.52% 98.95%      0.01s  0.52%  compress/flate.hash4
     0.01s  0.52% 99.48%      0.01s  0.52%  image/png.filter
```

我们看到，main.fillPixel函数花费的CPU时间是最多的。

在堆栈上发现main.paint函数并不奇怪，这是程序内部的函数。它是按像素输出的。但是，是什么导致paint函数花这么长时间呢？我们可以用cummulative flag（累加时间）进一步检查：

```golang
(pprof) top --cum
Showing nodes accounting for 1630ms, 85.34% of 1910ms total
Showing top 10 nodes out of 35
      flat  flat%   sum%        cum   cum%
         0     0%     0%     1840ms 96.34%  main.main
         0     0%     0%     1840ms 96.34%  runtime.main
     820ms 42.93% 42.93%     1630ms 85.34%  main.fillPixel
         0     0% 42.93%     1630ms 85.34%  main.seqFillImg
     810ms 42.41% 85.34%      810ms 42.41%  main.paint
         0     0% 85.34%      210ms 10.99%  image/png.(*Encoder).Encode
         0     0% 85.34%      210ms 10.99%  image/png.Encode
         0     0% 85.34%      160ms  8.38%  main.(*img).At
         0     0% 85.34%      160ms  8.38%  runtime.convT2Inoptr
         0     0% 85.34%      150ms  7.85%  image/png.(*encoder).writeIDATs
```

**以上输出显示实际上是main.fillPixed函数做了大部分的工作**（根据累加耗时排序后，除系统自带的函数外，main.fillPixel函数的累计值是最多的)。

## 4.4 Tracing VS Profiling

希望这个例子已经说明了监控分析的局限性。**监控分析告诉我们监控分析器看到的整体的视图：fillPixel正在做所有的工作。但我们看不到fillPixel为什么慢，哪里最耗时**。

那么现在，我们来介绍**执行跟踪器**：它从另一个不同的角度来分析该程序。

### 4.4.1 使用执行跟踪器

```golang
import "github.com/pkg/profile"

func main() {
	defer profile.Start(profile.TraceProfile, profile.ProfilePath(".")).Stop()
}
```

当我们执行这个程序时，我们在当前目录会得到一个trace.out文件。

```shell
% go build mandelbrot.go
% time ./mandelbrot
2017/09/17 13:19:10 profile: trace enabled, trace.out
2017/09/17 13:19:12 profile: trace disabled, trace.out

real    0m1.740s
user    0m1.707s
sys     0m0.020s
```

跟pprof一样，在go命令行中有一个工具可以分析这个文件：**go tool trace**

```shell
% go tool trace trace.out
2017/09/17 12:41:39 Parsing trace...
2017/09/17 12:41:40 Serializing trace...
2017/09/17 12:41:40 Splitting trace...
2017/09/17 12:41:40 Opening browser. Trace viewer s listening on http://127.0.0.1:57842
```

如下图是执行结果：

图-高性能4-1.png

go tool trace和 go tool pprof有点不同。执行跟踪器复用了很多chrome浏览器内置的可视化基础组件，所以 go tool trace扮演了一个服务器，将原始的跟踪信息转换成了chrome浏览器可以渲染的数据。

### 4.4.2 分析跟踪数据

我们首先点击 “view trace”连接进入对应的页面，如下图。

图-高性能4-2-cpu个数.png

从跟踪信息中我们看到该程序只用了一个CPU，如图红框中，代表一个有4个虚拟处理器，但只有红框中的1个在使用。

```golang
func seqFillImg(m *img) {
	for i, row := range m.m {
		for j := range row {
			fillPixel(m, i, j)
		}
	}
}
```

这个并不奇怪，默认情况下，mandelbrot.go依次为每一行中的每个像素顺序调用fillPixel函数。

一旦图片被渲染，执行跟踪器将切换到写入.png文件。这会在堆内存上产生碎片垃圾，所以在这一点上跟踪信息发生了变化，我们看到了经典的垃圾回收模式。

**trace分析提供了微秒级别的时序图**。这是我们用pprof工具所不能获取到的。

> ***go tool trace***
> 在我们继续之前，我们先讨论下关于go tool trace工具的一些用法
> + 该工具使用Chrome浏览器内置的javascript调试器。trace profile只能在Chrome浏览下工作，在Firefox，Safari，IE下是不能正常运行的。
>
> + 因为是Google的产品，所以它支持使用快捷键。例如用 ***WASD*** 导航，用 *** ？***获取列表
> + 查看跟踪图会消耗很多的内存。说实话，4Gb不会觉得多，8GB可能是最小值，越多越好。

## 4.5 使用更多的CPU

从上面我们知道，跟踪的程序是顺序执行的，而且并没有利用到多核CPU的优势。

曼德勃罗图的生成是可以并发执行的。每个像素都相互独立，他们可以并行的计算。所以，让我们试试用多个cpu执行。

```golang
% go build mandelbrot.go
% time ./mandelbrot -mode px
2017/09/17 13:19:48 profile: trace enabled, trace.out
2017/09/17 13:19:50 profile: trace disabled, trace.out

real    0m1.764s
user    0m4.031s
sys     0m0.865s
```

而且这个执行时间基本和单核cpu是一样的。但是user time却多了点，这是有道理了，因为我们使用了所有的CPU，但real时间（wall time）大致相同。

让我们再来看看trace
正如你看到的，这次trace产生了更多的数据：
图trace-px.png

+ 看起来很多工作正在完成，但是如果您放大再放大，就会发现每个协程运行的时间很多，之间会有空隙。这是由于程序调度而产生的。
+ 我们使用了4个CPU核心同时工作，因为每个fillPixel是执行了一个相对较小的工作量，所以我们会花费大量的调度时间。

## 4.6 批量处理

每个像素使用一个goroutine粒度太细了。没有足够的工作来证明goroutine的成本是合理的。

相反，让我们试试用一个goroutine来处理一行像素：

```golang
% go build mandelbrot.go
% time ./mandelbrot -mode row
2017/09/17 13:41:55 profile: trace enabled, trace.out
2017/09/17 13:41:55 profile: trace disabled, trace.out

real    0m0.764s
user    0m1.907s
sys     0m0.025s
```

这看起来是一个非常好的改进，我们几乎节省了一半的运行时间。让我们来看看跟踪信息。

图高性能4-trace-row.png

正如你所看到的，trace现在比之前更小更容易查看了。我们可以看到整个trace，这是个不错的改进。

+ 在程序刚开始的地方，我们看到协程的数量在1000个左右，这比上面每个像素使用一个协程的程序产生的1 << 20个协程是一个很大的改进。
+ 放大跟踪图，我们可以看到每个onePerRowFillImg运行的时间会更长，同时协程的生成工作提前完成，所以调度器可以有效的处理剩余的可运行的协程。

## 4.7 使用workers模式

mandelbrot.go支持另一种模式，让我们试试。

```shell
% go build mandelbrot.go
% time ./mandelbrot -mode workers
2017/09/17 13:49:46 profile: trace enabled, trace.out
2017/09/17 13:49:50 profile: trace disabled, trace.out

real    0m4.207s
user    0m4.459s
sys     0m1.284s
```

那么，运行时间比之前的更多了。让我们来跟踪并看看能否找到发生了什么。

图高性能4-trace-worker-1.png

查看trace，你会看到只有1个生产者和1个消费者在来回切换，因为在我们的程序中只有1个生产者和1个消费者。那让我们增加下workers的数量：

```shell
% go build mandelbrot.go
% time ./mandelbrot -mode workers -workers 4
2017/09/17 13:52:51 profile: trace enabled, trace.out
2017/09/17 13:52:57 profile: trace disabled, trace.out

real    0m5.528s
user    0m7.307s
sys     0m4.311s
```

花费的时间更多了。更多的real时间，更多的cpu时间。让我们继续trace看看发生了什么。

这个跟踪是比较混乱的。现在有很多的可用workers，但好像是所有时间都花在了他们竞争工作上。

**这是因为使用了无缓冲channel。一个无缓冲channel直到有接受者接收信息的时候才会发送消息**。

+ 直到有个一个消费worker准备好接收信息时，生产者才能发送消息到通道
+ 消费者worker直到生产者准备好发送数据的时候才会接收，所以他们之间是在相互等待各自完成工作。
+ 发送者没有特权，它不能优先于已经运行的消费者worker

我们在这里看到的是无缓冲channel带来的大量延迟。在调度器里有很多开启、暂停，本质上是相互等待时产生的加锁和互斥，这就是我们看到的sys时间耗时多的原因。

## 4.8 使用缓冲channel

```golang

import "github.com/pkg/profile"

func main() {
	defer profile.Start(profile.TraceProfile, profile.ProfilePath(".")).Stop()
```

```shell
% go build mandelbrot.go
% time ./mandelbrot -mode workers -workers 4
2017/09/17 14:23:56 profile: trace enabled, trace.out
2017/09/17 14:23:57 profile: trace disabled, trace.out

real    0m0.905s
user    0m2.150s
sys     0m0.121s
``` i hu

这和上面的每行一个协程的模式花费的时间非常接近。

使用缓冲channel后，trace会展示给我们如下信息：
klutgv 冲突个u'tg
+ 生产者不需要等待消费worker，它可以直接填充到channel中。
+ 消费worker不需要等待生产者产生数据就可以直接从channel中获取到下一条数据。

使用带缓冲的channel我们达到了和每行一个协程的同样的性能。
vhchghg g dggdggghj ggg
## 4.9更多资源rtr

+ Rhys Hiltner, [Go’s execution tracer](https://www.youtube.com/watch?v=mmqDlbWk_XA) (dotGo 2016)
+ Rhys Hiltner, [An Introduction to "go tool trace"](https://www.youtube.com/watch?v=V74JnrGTwKA) (GopherCon 2017) h vh    ghnch  j f hv1Aw     1             111!11111111111111111111111111111 MB JHG
+ Dave Cheney, [Seven ways to profile Go programs](https://www.youtube.com/watch?v=2h_NFBFrciI) (GolangUK 2016)
+ Dave Cheney, [High performance Go workshop](https://dave.cheney.net/training#high-performance-go)
+ Ivan Daniluk, [Visualizing Concurrency in Go](https://www.youtube.com/watch?v=KyuFeiG3Y60) (GopherCon 2016)
+ Kavya Joshi, [Understanding Channels](https://www.youtube.com/watch?v=KBZlN0izeiY) (GopherCon 2017)
+ hgn                             nbdmhv HHVGHHJFHT HTGY V59O KLUFT U, UVVVVN 