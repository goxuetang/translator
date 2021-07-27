# Go高性能系列教程之读懂Go的执行跟踪器

## 介绍

有没有想过你的协程是如何被go运行时调度的？有没有想过为什么增加了并发但性能却没有提高？***Go的执行跟踪器（execution tracer）可以帮我们回答这些 问题以及帮你诊断性能问题，比如延迟，竞争和并行阻塞***。

该工具自Go 1.15版本开始可用，主要工作是收集go运行时特定的事件。例如：

- 协程的创建、启动以及结束。
- 协程阻塞/解除阻塞事件（系统调用syscalls，通道channels，锁locks）
- 网络I/0相关事件
- 系统调用
- 垃圾回收（GC）

所有这些数据都是通过跟踪器（tracer）进行收集的，没有任何聚集或取样。在一些繁忙的应用中，可能会导致一个非常大的文件，该文件可以通过 **go tool trace**命令进行分析。

在Go的工具链中，之前已经有pprof可以收集内存和CPU的数据了，为什么还要研发执行跟踪器，并把它加入到官方的工具链中呢？然而，CPU的收集器（CPU Profiler）能够告诉你一个函数花费了多少CPU的执行时间，但不能告诉你在运行时环境中，是什么阻塞了协程的运行，也不能告诉你协程在可用的线程上是如何调度器的。这就是跟踪器（tracer）真正发挥作用的地方。[tracer设计文档](https://docs.google.com/document/u/1/d/1FP5apqzBgr7ahCCgFO-yoVhk4YZrNIDNf9RybngBc14/pub)很好的解释了tracer的动机以及如何被设计的。


## 开启tracer之旅
我们以"Hello World"来作为执行跟踪（tracing）的示例。在这个例子中，我们使用runtime/trace包中的start/stop方法来讲跟踪的数据写入到标准错误输出中。跟踪输出将被写到标准的错误输出中--os.stderr
```golang
 package main
 
 import (
 	"os"
 	"runtime/trace"
 )
 
 func main() {
 	trace.Start(os.Stderr)
 	defer trace.Stop()
 	//create new channel of type int
 	ch := make(chan int)
 	
 	// start new anonymouse goroutine
 	go func() {
 		// send 42 to channel
 		ch <- 42
 	}()
 	
 	// read from channel
 	<-ch
 }
```

使用 **go run main.go 2> trace.out** 命令运行示例代码，并将跟踪数据输出到trace.out文件中，稍后使用**go tool trace trace.out** 进行查看分析。

运行该命令后，一个浏览器窗口会被打开，页面上有一些选项。每一个选项都会打开一个不同的跟踪视图，包含程序执行的不同信息。

图：trace-opt.png

#### View trace
最复杂、最强大、最具交互性的可视化显示了整个程序执行的时间表。例如，该视图显示了在每个虚拟处理器上正在在执行的内容以及在等待运行时被阻塞的内容。稍后我们会深入探讨这个可视化内容。

#### Goroutine analysis
该选项展示了在整个运行期间，每种协程有多少个被创建。在选中了一种协程后，可以看到该种类的每个协程的信息。例如，当一个协程试图获取互斥资源的锁的时候，从网络读取数据的时候以及运行的时候被阻塞了多长时间。

#### Network/Sync/Syscall blocking profile
这些内容以图表的方式展示了在这些资源上阻塞协程花费了多长时间。它们和pprof工具的内存 profiler和cpu profiler的图形非常相似。例如，这里是排查锁竞争最好的地方。

#### Scheduler latency profiler
提供调度程序级别信息的计时，显示调度时间最多的地方。

## View Trace
点击“View trace”链接，是关于整个程序执行的视图。

下图中高亮的部分是该视图最重要的部分，每一个模块在下面有详细描述：
图：view-trace.png

1. Timeline
展示了在执行期间的时间，并且通过导航按钮可以改变时间单位。通过键盘的快捷键（WASD键，像视频游戏）可以左右移动、放大缩小时间轴线。

2. Heap
展示了在执行期间的内存分配，此项对发现内存峰值以及检查在每次运行后GC能够释放多少内存是非常有用。

3. Goroutines
展示了哟多少协程正在运行，以及在每个时间点有多少协程正等待被调度运行。如果带运行的协程数量过大，则代表调度阻塞。例如，当程序创建了很多协程导致调度器压力很大时就会发生此情况。

4. OS Threads
展示了有多少OS线程正在被使用以及多少被syscalls调用阻塞。

5. 虚拟处理器
每个虚拟处理器一行。虚拟处理器的数量受**GOMAXPROCS**环境变量的控制（默认是CPU的核心数）

6. Goroutines和events
展示了在每个虚拟处理器上都运行了什么地方的哪些协程。协程之间的连线代表事件。在该示例图中，我们可以看到协程“G1runtime.main”拷贝出两个不同的协程：G6和G5(前一个协程是收集跟踪数据的协程，后一个协程是我们用go关键词启动的协程）。

每一个处理的第二行展示了额外的事件例如syscalls和运行时事件。这还包括goroutine代表运行时所做的一些工作（例如协助垃圾收集器）。

下面这张图显示了当选择了一个特定的协程后所收集到的信息。

图：view-goroutine.png

包括以下信息：
+ 协程的名称（Title）
+ 开始的时间（Start）
+ 持续的时间（Wall Duration）
+ 开始时栈跟踪信息（Start Stack Trace）
+ 结束时栈跟踪信息（End Stack Trace）
+ 该协程产生的事件（Events）

我们可以看到，该协程创建了两个时间：跟踪器协程和往channel上发送42数据的协程。

图：view-event.png

点击一个特定的事件（图上的一行或点击协程后选择某个特定的事件），我们会看到：
+ 当时间开始的时候的栈跟踪信息
+ 事件的持续时间
+ 事件中参与的协程

点击某个协程可以跳转到该协程的追踪数据页面。

## Blocking profiles
在一次跟踪中，另外一个特殊的视图是network/synchronization/syscall阻塞分析。阻塞分析显示了一个类似pprof中内存/cpu分析的图表。该图表上展示的不是每次函数调用分配的内存，而是在特定资源上，每个协程在阻塞上花费了多长的时间。

下面这张图显示了我们代码的“Synchronization blocking profile”。

图四：blocking-profile.png

该图告诉我们 main 协程在从channel中接收数据时阻塞了12.08毫秒。当许多协程在竞争获取一个共享资源的锁的时候，这个图是可以发现锁竞争的非常好的方式。

## Collecting Traces

有三种方式可以收集跟踪信息：
1. 使用 *runtime/trace* 包
这种方式需要调用 trace.Start 和 trace.Stop，在我们的“Hello World”的例子中使用的就是这种方式。
2. 使用 -trace=\<file\> test flag
对于在测试期间的代码来说这是最有用的方式
3. 使用debug/pprof/trace handler
对于正在运行的web应用程序来说这是最好的方式。

## 跟踪web应用程序

在一个正在运行的web应用程序中，为了能够收集跟踪数据，需要添加 /debug/pprof/trace处理器。下面的代码演示了使用 http.DefaultServeMux默认处理器：导入 net/http/pprof包即可。

```golang
package main

import (
	"net/http"
	_ "net/http/pprof"
)

func main() {
	http.Handle("/hello", http.HandlerFunc(helloHandler))
	http.ListenAndServe("localhost:8181", http.DefaultServeMux)
}

func helloHandler(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("hello world!"))
}
```

为了能够收集跟踪数，我们需要给该节点发送一个请求，例如 *curl localhost:8181/debug/pprof/trace?seconds=10 > trace.out*。该请求将会阻塞10秒，然后跟踪数据被写入到trace.out文件中。然后就可以通过 *go tool trace trace.out* 命令来查看分析跟踪数据了。

> 安全提示：注意将pprof的处理器暴露在互联网上是不可取的。建议在仅绑定到本地可访问不同的http.Server上。[该博客](http://mmcloughlin.com/posts/your-pprof-is-showing)讨论了这些风险，并提供了有关如何正确处理pprof handler的代码示例
> 

在收集跟踪数据之前，先使用 wrk 工具给我们的服务制造一些压力：

```bash
$ wrk -c 100 -t 10 -d 60s http://localhost:8181/hello
```
这将同时使用10个线程构造100个连接持续请求60秒。当 *wrk* 运行的时候，我们可以收集5秒的跟踪数据： curl localhost:8181/debug/pprof/trace?seconds=5 > trace.out。这在我4核心的CPU机器上缠身过了5MB的文件。

再次使用 *go tool trace trace.out* 命令来分析跟踪文件。因为要解析整个文件，这将比之前示例中花费更多的时间。当解析完成后，页面看起来跟之前有点不一样：
```bash
View trace (0s-2.546634537s)
View trace (2.546634537s-5.00392737s)

Goroutine analysis
Network blocking profile
Synchronization blocking profile
Syscall blocking profile
Scheduler latency profile
```
为了保证浏览器能够渲染所有的东西，此工具已经将跟踪数据自动分隔成了两个连续的部分。更繁忙的应用程序或更长到的跟踪时间可能需要被拆分成更多的部分。

点击 “View trace（2.546634537s-5.00392737s）”我们看到发生了很多事情：

图五:trace-web.png

这部分截屏显示了在1169毫秒到1170毫秒之间开始运行的GC，以及在1174毫秒之后结束。在此期间，一个OS线程（proc1）运行一个专门用于GC的协程，而其他协程在某些GC阶段会提供辅助（这些会在goroutine行中显示，并被有“辅助标记”）。在结束结束的位置，我们看到，分配的内存大部分是由GC释放的。

另一个有用的信息是标记有“Runnable”（可运行）状态的协程的数量（在选中的时间段上有13个）：如果该值过大，则意味着我们需要更多的CPU来处理负载。

## 结论

跟踪（tracer）对于调试并发问题是一个强大的工具，例如竞争（contentions and logical races）。但它并没有解决所有的问题：解决不了哪段代码花费的CPU时间长或者分配的内存多的问题。对于这种场景，*go tool pprof* 是更好的选择。

这个工具真正的用处在于当你想了解该程序在运行这段时间内都做了什么事情以及想知道当没有协程在运行时每个协程正在做什么的场景。收集跟踪可能会有一定的开销，并且会产生大量的检查数据。

不幸的是，官方文档比较匮乏，所以一些实验需要尝试并理解跟踪器正在展示的内容。当然这也是一个给社区官方文档贡献的好机会。














