# 如何避免Go中常见的错误

## 一、 简介
内容来源：

#### 观点一：Go易学难精。
易学因为简单：
- Go没有类型继承，没有异常，没有宏，没有运算符重载。
- Go只有三种标准的数据结构类型：array，slice，map。
- Go中总共直有25个关键词。
- 起协程很简单。go关键词

难以精通因为：
- 什么时候使用接口？
- 什么时候使用值接收，什么时候使用指针接收？
- 如何高效处理切片？
- 如何干净而富有表现力的处理错误管理？
- 如何避免内存泄露？
- 如何编写相关测试和基准测试？

#### 观点二：从错误中学习更有效。

错误，不仅仅可以让你记住错误本身，而且能够让你记住错误发生时的上下文。

所以本书是从**基础知识**、**代码组织**、**数据和控制结构**、**字符串**、**函数和方法**、**错误管理**、**并发**、**测试**、**优化和生产**。

## 二、案例

- 变量隐藏（Variable shadowing）
- 比较值（Comparing values）
- 防止常见的JSON错误
- 处理枚举
- 使用defer并理解如何处理参数
- 释放资源

### 变量隐藏

在该例中，我们使用两种不同的方式创建一个HTTP客户端，具体取决于tracing布尔值：

```golang
var client *http.Client	①
if tracing {
	client, err := createClientWithTracing()	②
	if err != nil {
		return err
	}
	log.Println(client)
}else {
	client, err := createDefaultClient()	③
	if err != nil {
		return err
	}
	log.Println(client)
}

//use client
```
#### 问题1：请问 第一行的client变量值最终是什么？





#### 问题2：我们如何确保给client赋值了呢？有两种不同的方法。



**方法一：**  在内部块中使用临时变量，像下面这样：

```golang
var client *http.Client
if tracing {
	c, err := createClientWithTracing()	①
	if err != nil {
		return err
	}
	client = c ②
} else {
	c, err := createDefaultClient()
	if err != nil {
		return err
	}
	client = c
}
// Use client
```
① 创建了一个临时变量c
② 将临时变量赋给变量client
变量c的生命周期只在if/else块中。然后，我们将这些变量赋值给client。

**方法二：** 在内部块中使用赋值操作符（=）来将函数的返回值直接赋值给client变量。然而，它需要创建一个error变量，因为赋值运算符仅在已声明变量时才起作用。

```golang
var client *http.Client
var err error	①
if tracing {
	client, err = createClientWithTracing() ②
	if err != nil {
		return err
	}
} else {
	client, err = createDefaultClient()
	if err != nil {
		return err
	}
}
```
① 声明变量err
② 使用赋值操作符将返回来的*http.Client直接赋值给client变量



### 当在内部块中将一个变量名重新声明时就会发生变量隐藏。

#### 问题3：如何检测隐藏变量呢？
- 使用linters工具：vet & shadow
```golang
$ go install golang.org/x/tools/go/analysis/passes/shadow/cmd/shadow 
go: finding golang.org/x/tools latest
go: downloading golang.org/x/tools v0.0.0-20201218024724-ae774e9781d2
go: extracting golang.org/x/tools v0.0.0-20201218024724-ae774e9781d2
$ go vet -vettool=$(which shadow) 
./main.go:8:3: declaration of "i" shadows declaration at line 6
```
① shadow安装
② 使用vettol参数将shadow链接到vet
③ Go vet可以检测隐藏的变量了

```golang
package main

import "fmt"

func main() {
    i := 0
    if true {
        i := 1 ①
    }
    fmt.Println(i)
}
```
① 隐藏的变量


### 比较值（Comparing values）

首先创建一个customer结构体，并用 == 操作符来比较两个实例。
```golang
type customer struct {
	id string
}

func main() {
	cust1 := customer{id: "x"}
	cust2 := customer{id: "x"}
	fmt.Println(cust1 == cust2)
}
```

#### 问题1： 代码输出什么？true or False？。

现在，如果我们对customer结构体稍微做下修改，在其中加入一个slice的字段，那将会发生什么：
```golang
type customer struct {
	id string
	operations []float64 ①
}

func main() {
	cust1 := customer{id: "x", operations: []float64{1.}}
	cust2 := customer{id: "x", operations: []float64{1.}}
	fmt.Println(cust1 == cust2)
}
```
① 新加入的字段

#### 问题2：代码的输出又是什么？True or False ?

编译错误：
```bash
invalid operation: cust1 == cust2 (struct containing []float64 cannot be compared)
```

#### 该问题和 == 和 != 操作符的工作原理有关。如果两种类型具有可比较性，那么我们可以使用这两种运算符来比较两种不同的类型。

在Go中可比较的类型包括：
- 布尔值： == 和 != 可以比较两个布尔类型的值是否相等
- 数字：== 和 != 可以比较两个数字类型的值是否相等。如果两个值具有相同的类型或能够转成成相同的类型，那么这两个操作也是可以正常编译的。
- 字符串：== 和 != 可以比较两个字符串是否相等。我们可以根据字符串的词序使用>=, < 和 > 操作符对两个字符串进行比较。
- 指针： == 和 != 可以比较两个指针是否指向了相同的内存地址或者是否都是nil。
-  通道（channels）：== 和 != 可以比较两个通道是否是由同一个make创建的或者两个都是nil

**如果struct和array仅有可比较的类型组成，我们也可以将他们添加到此列表中**。

在第一个版本中，customer结构体是由一个单一的可比较类型（一个字符串）组成的，所以使用==进行比较是合法的。相反，在第二个版本中，因为结构体customer中包含了一个slice，无法使用 == 运算符进行比较并导致了编译错误。

我们还应该知道在interface{}类型中使用 == 和 !=操作符可能存在的问题。此外，它将导致一个运行时panic：
```bash
panic: runtime error: comparing uncomparable type main.customer
```


#### 问题3：如何对slice、map、或者包含不能比较类型的struct进行比较？

一种是使用标准库。另外一种是使用reflect和reflect.DeepEqual。该方法对两个元素进行深度比较。该函数接受的参数是基本类型，数组，结构体，切片（slice），map，指针，接口和函数。

让我们再返回第一个例子中，这次使用reflect.DeepEqual：
```golang
cust1 := cutomer{id: "x", operations: []float64{1.}}
cust2 := customer{id: "x", operations: []float64{1.}}
fmt.Println(reflect.DeepEqual(cust1, cust2))
```
这里，即使在customer结构体中包含不可比较的类型（slice），但依然会如期望的那样进行操作并输出true。

在使用reflect.DeepEqual函数的时候，需要注意两点。
1. **该函数区分了空集合和零值**。

现在比较两个customer结构体，其中一个包含nil切片，而第二个是空切片：

```golang
var cust1 interface{} = customer{id: "x"} ①
var cust2 interface{} = customer{id: "x", operations: []float64{}} ②
fmt.Print("%t\n", reflect.DeepEqual(cust1, cust2))
```
① Nil切片
② 空切片

**输出：false**
因为reflect.DeepEqual函数认为空和nil集合是不同的。

2. **性能**。由于此函数使用反射，因此会有性能方面的损耗。一般来说说，reflect.DeepEqual的平均执行速度要比 == 操作符慢***100*** 倍。



### 防止常见的JSON错误


#### 1、空JSON
首先，定义一个point结构体：
```golang
type point struct {
	x float32
	y float32
}
```

让我们创建一个point示例并使用标准的json.Marshal函数把该实例编码成一个JSON输出：
```golang
p := point{3., 2.5}
b, err := json.Marshal(p) ①
if err != nil {
	return err 
}
fmt.Println(string(b)) ②
```
① Marshal p
② b是一个[]byte变量，我们需要把它转换成可读的字符创。

#### 该示例输出什么？
```bash
{}
```

**然后，指定json标签，又输出什么？**
```golang
type point struct {
	x float32 `json:"x"` ①
	y float32 `json:"y"` ②
}
```
① 设置x的JSON标签
② 设置y的JSON标签

**输出仍然为空：{ }。
说明JSON标签不是必须的。默认情况下，JSON字段的名称会和结构体字段的名称相同**。

***我们将point结构体名称重新命名为 Point，将结构体的可见性公开**：
```golang
type Point struct { ①
	x float32
	y float32
}
```

**输出依然是空：{ }。所以，对于marshaled/unmarshaled来说，结构体也不一定要被导出**。

**接着尝试，将结构体的内部字段x、y字段名改成大写X、Y**：
```golang
type point struct {
	X float32 ①
	Y float32 ②
}
```

输出结果：
```bash
{"x": 3, "Y": 2.5}
```
#### 结论1：要进行json的marshaled/unmarshaled，结构体的字段必须被导出。







#### 那么， 在marshaling中该如何忽略一些字段呢？例如，我们可能想要忽略掉password字段。

**方法一：我们不导出这些字段**。 
**方法二：使用JSON的 -  标签，将已导出的字段忽略掉**。

示例：
```golang
type Foo struct {
	A string
	b string ①
	C string `json:"-"` ②
}

f := Foo{
	A: "a",
	b: "b",
	C: "c",
}
b, err := json.Marshal(f)
if err != nil {
	return err
}
fmt.Println(string(b))
```
① 该字段被忽略是由于没被导出，即作用域是私有变量
② 该字段被忽略是因为使用了JSON的标签 "-"

因为字段b没有被导出，字段C被强制忽略了，所以将struct经过marshaled后，只包含字段A：
```json
{"A": "a"}
```
#### 小结
- 要想在marshaled/unmarshaled中输出，相关的字段必须要被导出。
- 我们不必将整个结构体类型导出或者强制使用JSON标签。
- 我们也可以通过将字段不导出或者使用指定的JSON标签来忽略相关的字段。

在下一节中，我们将看到一个处理匿名字段时的常见错误。


#### 2、缺失字段名在json编解码的时候会是什么样？
在Go语言中，如果我们声明了一个没有名称的字段，这叫做嵌入字段。嵌入字段用于提升嵌入类型的字段和方法，示例如下：
```golang
type Event struct {
	ID int
	time.Time ①
}
```

time.Time是一个嵌入字段，因为它没有名称声明。如果我们创建一个Event结构体类型，我们可以在Event结构体层直接访问time.Time的方法。
 ```golang
 event := Event{}
 second := event.Second() ①
```
 ① 如果结构体中没有嵌入time.Time类型，例如我们在上面的结构体中指定的是一个t变量名的字段，我们要访问Second方法时需要使用下面的方法：event.t.Second()

该Second方法被提升为可通过Event结构直接访问的方法。这就是为什么结构体和接口类型主要被应用于嵌入式字段，而不是像int或string之类的基本类型。

使用JSON的marshaling方法封装嵌入字段会有什么影响呢？让我们在下面的例子中找到答案。我们将实例化一个Event示例并把他marshal成JSON。下面的这段代码将输出什么呢？
 ```golang
 event := Event{
 	ID: 1234,
 	Time: time.Now(), ①
 }
 b, err := json.Marshal(event)
 if err != nil {
	return err
 }
 fmt.Printf("json: %s\n", string(b))
 ```
 ① 在实例化一个结构体的时候匿名字段的名称就是嵌入结构体的名字

 我们期望该代码输出一下信息：
```shell
 {"ID":1234,"Time":"2021-04-19T21:15:08.381652+02:00"}
 ```
 相反，它实际输出是这样的：
 ```shell
"2020-12-21T00:08:22.81013+01:00"
```
#### 为什么ID字段没有被输出呢？对于ID字段和其对应的1234值发生了什么？当这个字段被导出时，它应该已经被marshaled。




要理解这个问题，我们必须澄清两件事情。

首先，如果一个嵌入字段类型实现了一个接口，那么包含该嵌入字段的结构体也将实现这个接口。从某种意义上来说，它继承了一种能力。让我们看一下下面的例子，我们定义了一个Foo结构体和一个嵌入Foo字段的Bar结构体：

```golang
type Printer interface {  ①
	print()
}
type Foo struct{}

func (f Foo) print() { ②
	fmt.Println("foo")
}
type Bar struct {
	Foo ③
}
```
① 定义了一个Printer接口和一个print()方法
② 让Foo结构体实现了Printer
③ 在Bar结构体中嵌入Foo类型

这里，由于Foo实现了Printer接口，同时Foo是Bar结构体的一个嵌入字段，所以Bar也是一个Printer类型。我们可以实例化一个Bar类型并直接调用print方法，而不通过Foo字段。

其次，我们可以通过构造一个实现了json.Marshaler接口的类型来覆盖掉默认的marshaling行为。该接口包含一个唯一的MarshalJSON方法：
```golang
type Marshaler interface {
	MarshalJSON() ([]byte, error)
}
```

下面是我们自定义marshaling的一个例子：
```golang
type foo struct{} ①
func (_ foo) MarshalJSON() ([]byte, error) { ②
    return []byte(`"foo"`), nil ③
}
func main() {
    b, err := json.Marshal(foo{}) ④
    if err != nil {
			panic(err) 
		}
    fmt.Println(string(b))
}
```
① 定义结构体
② 实现MarshalJSON方法
③ 返回一个静态响应
④ json.Marshal方法将会依赖于自定义的MarshalJSON实现

因为我们已经覆盖了JSON的marshaling行为，所以这段代码将打印出 foo。

已经澄清了这两点，再让我们回到上面用Event结构体输出有问题的例子。我们已经知道了time.Time已经实现了json.Marshaler接口。由于time.Time是Event结构体的嵌入字段，提升了time.Time的方法到Event层级。因此，Event也实现了json.Marshaler方法。

因此，当我们传递Event到json.Marshal方法时，它不会使用默认的marshaling行文而是time.Time中提供的。这就是为什么在marshaling一个Event时导致忽略了ID字段的原因。

#### 要解决该问题，主要有两种可能的方法。

首先，我们可以给嵌入字段time.Time命名，即不使用嵌入字段：
```golang
type Event struct {
	ID int
	Time time.Time ①
}
```
① time.Time现在不是嵌入字段了

这样，如果我们marshal该版本的Event结构体时，它将打印出如下信息：
```bash
{"ID":1234,"Time":"2020-12-21T00:30:41.413417+01:00"}
```

如果我们想保留或必须保留该time.Time为嵌入字段，另外一种选择是让Event实现json.Marshaler接口。然而，该解决方法更麻烦，而且需要确保该MarshalJSON方法始终与Event结构保持同步。

我们应该小心使用嵌入字段。虽然嵌入类型提升了其字段和方法有时会很方便，但同时也能导致细微的bug，因为父结构体也隐式的实现了该接口。当使用嵌入字段时，我么应该确保了解可能的副作用。

#### 3 JSON和单调时钟
操作系统会处理两种不同的时钟类型：墙上时钟（wall）和单调时钟（monotonic）。在本节中，我们将会看到当time.Time和JSON一起使用时可能产生的影响，并了解为什么这种时钟差异对于理解至关重要。

在下面的例子中，我们将继续使用Event结构体，但是只包含一个time.Time字段（非嵌入字段）：
```golang
type Event struct {
	Time time.Time
}
```

我们将实例化一个Event，并将它marshal成JSON，并将JSON串unmarshal成另外一个结构体。然后，我们将比较这个两个结构体。让我们看看marshaling/unmarshaling过程是否总是对称的：
```golang
t := time.Now() ①
event1 := Event{ ②
	Time: t, 
}
b, err := json.Marshal(event1) ③
if err != nil {
	return err 
}
var event2 Event
err = json.Unmarshal(b, &event2) ④
if err != nil {
	return err 
}
isEquals := event1 == event2
```
① 获取当前本地时间
② 实例化一个Event结构体
③ Marshal成JSON
④ Unmarshaling JSON成结构体实例

#### 那么isEquals应该是什么值？它会是false，而非true。为什么？

首先，让我们打印event1和event2的内容：
```golang
fmt.Println(event1.Time) ①
fmt.Println(event2.Time) ②
```
① 2021-01-10 17:13:08.852061 +0100 CET m=+0.000338660
② 2021-01-10 17:13:08.852061 +0100 CET

所以，我们注意到打印的是两个不同的值。event1的值接近于event2的值，除了m=+0.000338660部分。那这部分是什么意思呢？

我们应该知道操作系统提供了两种时钟类型。首先，墙上时钟用于知道当天的当前时间。该时钟可能会发生变化。例如，如果使用NTP（网络时间协议）同步，时钟可以在时间上向后或向前跳跃。我们不应该使用墙上时钟来测量持续时间，因为我们可能会面临一些奇怪的行为，比如负的持续时间。这就是为什么操作系统提供第二种时钟类型的原因：**单调时钟**。单调时钟保证时间永远都是向前的，并且不会受时间跳跃的影响。它可能会受到潜在频率调整的影响（例如，如果服务器检测到本地石英的移动速度与NTP服务器不同），但不会受到时间跳跃的影响。

在Go中，并没有在两个不同的API来拆分两个类型的时钟，而是time.Time同时包含墙上时钟和单调时间。

当我们使用time.Now()获取本地时间时，会返回time.Time的两个时间：
```golang
 2021-01-10 17:13:08.852061 +0100 CET m=+0.000338660
------------------------------------ --------------
             Wall time               Monotonic time
```
相反，当我们解析JSON串时，time.Time字段并不包含单调时间，只有墙上时间。因此，当我们比较两个实例时，因为单调时间不一样而输出结果false。这也是我们在打印两个结构体时注意到差异的原因。

我们该如何解决这个问题呢？有两个主要选择。

当我们使用 == 操作符对两个time.Time进行比较时，它会比较结构体的所有字段包括，包括单调时间部分。为了避免这种情况，我们可以使用Equal方法作为替代：
```golang
areTimesEqual := event1.Time.Equal(event2.Time) ①
```
① true

Equal方法不会考虑单调时间。 因此，areTimesEqual会是true。然而，在这个例子中，我们只比较了time.Time字段，并没有比较它的父结构Event结构体。

看下Equal的实现：
```golang
func (t Time) Equal(u Time) bool {
	if t.wall&u.wall&hasMonotonic != 0 {
		return t.ext == u.ext
	}
	return t.sec() == u.sec() && t.nsec() == u.nsec()
}
```

第二个选择是依然保持使用 == 操作符来对两个结构体实例进行比较，但使用Truncate方法将单调时间替换成0值：

```golang
t := time.Now()
event1 := Event{
    Time: t.Truncate(0),
}
b, err := json.Marshal(event1)
if err != nil {
	return err
}
var event2 Event
err = json.Unmarshal(b, &event2)
if err != nil {
	return err 
}
isEquals := event1 == event2
```
① 去除单调时间
② 使用 == 操作符对两个结构体实例进行比较

在此版本中，这两个time.Time字段是相等的。因此，isEquals将会返回true。

总而言之，marshaling/unmarshaling处理的程序并不总是可逆的，我们会遇到在结构体中包含time.Time字段的场景。例如，我们应该牢记这一原则，以免编写错误的测试。

