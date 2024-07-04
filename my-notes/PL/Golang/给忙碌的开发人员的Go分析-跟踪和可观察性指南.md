
## 介绍

### Read this

这是⼀本实⽤指南，针对的是对使⽤剖析、跟踪和其他可观察性技术改进程序感
兴趣的忙碌的 gopher。如果你对 Go 的内部实现不熟悉，建议你先阅读整个介
绍。之后，你可以⾃由地跳到你感兴趣的任何章节。

### Go心智模型

虽然不了解 Go 语⾔的底层原理，但是你也能熟练地编写 Go 代码。但是当涉及到性能和调试时，你会从内部实现中受益匪浅。我们接下来会概述 Go 基本原理的模型。这个模型应该⾜以让你避免最常⻅的错误，但是[所有的模型都是错误的](https://en.wikipedia.org/wiki/All_models_are_wrong)，所以我们⿎励你寻找更深⼊的材料来解决未来的难题。

Go 的主要⼯作是对硬件资源进⾏复⽤和抽象，类似于操作系统。主要使⽤两个抽
象：

1. Goroutine Scheduler(goroutine 调度器): 管理你的代码如何在 CPU 上执⾏。
2. Garbage Collector(垃圾回收): 提供虚拟内存，在需要时⾃动释放。
#### Goroutine调度器

让我们先⽤下⾯的例⼦来谈谈调度器:

```go
func main() {
	res, err := http.Get("https://example.org/")
	if err != nil {
		panic(err)
	}
	fmt.Printf("%d\n", res.StatusCode)
}
```

这⾥我们有⼀个 goroutine，我们称之为 G1 ，它运⾏ main 函数。下图显示了这个 goroutine 如何在单个 CPU 上执⾏的简化的时间线。最初 G1 在 CPU 上运⾏以准备 http 请求。然后 CPU 变得空闲，因为 goroutine 需要⽹络等待。最后它再次被调度到 CPU 上，打印出状态代码。

![[Pasted image 20240325113148.png]]

从调度器的⻆度来看，上述程序的执⾏情况如下所示。⼀开始， G1 在 CPU 1 上Executing 。然后，在 Waiting ⽹络的过程中，goroutine 被从 CPU 上取出。⼀旦调度器注意到⽹络已经返回（使⽤⾮阻塞 I/O，类似于 Node.js），它就把goroutine 标记为 Runnable 。⼀旦有 CPU 核⼼可⽤，goroutine 就会再次开始Executing 。在我们的例⼦中，所有的 cpu 核都是可⽤的，所以 G1 可以⽴即回到其中⼀个 CPU 上 Executing fmt.Printf() 函数，⽽⽆需在 Runnable 状态下花费任何时间。

![[Pasted image 20240325113239.png]]

⼤多数时候，Go 程序都在运⾏多个 goroutine，所以你会有⼀些 goroutine 在⼀些 CPU 核⼼上 Executing ，⼤量 goroutine 由于各种原因 Waiting ，理想情况下没有 goroutine Runnable ，除⾮你的程序显示⾮常⾼的 CPU 负载。下⾯就是⼀个例⼦。

![[Pasted image 20240325113315.png]]

当然，上⾯的模型掩盖了许多细节。Go 调度器是在操作系统管理的线程之上运⾏的，甚⾄ CPUs 本身也能够实现超线程，这可以看作是⼀种调度形式。所以如果你感兴趣的话，可以通过 [Ardan labs 的 Scheduling in Go]([Scheduling In Go : Part I - OS Scheduler (ardanlabs.com)](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)) 或其他资料继续深⼊这个主题。

但是，上⾯的模型应该⾜以理解本指南的其余部分。特别是，各种 Go profilers所测量的时间基本上是 goroutines 在 Executing 和 Waiting 状态中所花费的时间，如下图所示。

![[Pasted image 20240325113440.png]]

#### 垃圾收集器

Go 的另⼀个主要抽象是垃圾收集器。在 C 语⾔中，程序员需要使⽤ malloc()和 free() ⼿动分配和释放内存。这提供了很好的控制，但在实践中很容易出错。垃圾收集器可以减少这种负担，但内存的⾃动管理很容易成为性能瓶颈。本指南将为 Go 的 GC 提供⼀个简单模型，这个模型对于识别和优化内存管理相关问题⾮常有⽤。
##### The Stack

让我们从最基础开始。Go 可以将内存分配到堆栈或者堆的任意⼀个位置。每个
goroutine 都有⾃⼰的堆栈，栈是⼀个连续的内存区域。此外，goroutine 之间还
有⼀⼤块内存共享区域，这就是堆。如下所示。

![[Pasted image 20240325113558.png]]

当⼀个函数调⽤另⼀个函数时，它会在堆栈中获得⾃⼰的部分，称为 stack frame(堆栈帧) ，在这⾥它可以创建局部变量等。堆栈指针⽤于标识帧中的下⼀个空闲位置。当⼀个函数返回时，通过简单地将堆栈指针移回到前⼀帧的末尾，最后⼀帧中的数据就会被丢弃。帧的数据本身可以留在堆栈中，并被下⼀个函数调⽤覆盖。这是⾮常简单和有效的，因为 Go 不需要跟踪每个变量。

为了让这个更直观⼀点，让我们看看下⾯的例⼦:

```go
func main() {
	sum := 0
	sum = add(23, 42)
	fmt.Println(sum)
}

func add(a, b int) int {
	return a + b
}
```

我们有⼀个 main() 函数，它⼀开始就在堆栈中为变量 sum 创建了空间。当调⽤add() 函数时，它获得⾃⼰的帧来保存本地的 a 和 b 参数。⼀旦 add() 返回，通过将堆栈指针移回 main() 函数帧末尾，它的数据就会被丢弃， sum 变量就会得到更新。同时， add() 的旧值逗留在堆栈指针之外，以便被下⼀个函数调⽤覆盖。下⾯是这个过程的可视化图:

![[Pasted image 20240325113739.png]]

上⾯的例⼦是⾼度简化了，省略了许多关于返回值、帧指针、返回地址和函数内联的细节。实际上，在 Go 1.17 中，上⾯的程序甚⾄可能不需要堆栈上的任何空间，因为少量的数据可以由编译器使⽤ CPU 寄存器来管理。但这也没关系。这个模型应该还是能让你对简单的 Go 程序在堆栈上分配和丢弃局部变量的⽅式有⼀个直观认识。

此时你可能会想，如果堆栈上的空间⽤完了会怎么样。在像 C 这样的语⾔中，这会导致堆栈溢出错误。⽽ Go 会⾃动处理这个问题，扩容成两倍堆栈。所以⼀般goroutine 开始都是很⼩的，通常是 2 KiB，这也是 goroutine ⽐操作系统线程更具[可伸缩性](https://golang.org/doc/faq#goroutines)的关键因素之⼀。
##### The Heap

堆栈分配很 nice，但是在很多情况下 Go 不能全部都使⽤它们。最常⻅⽐如函数返回指针。把上⾯的 add() 函数示例修改⼀下，如下:

```go
func main() {
	fmt.Println(*add(23, 42))
}

func add(a, b int) *int {
	sum := a + b
	return &sum
}
```

通常 Go 可以把 add() 函数内部的 sum 变量分配到堆栈中。我们已经知道当add() 函数返回时，这些数据将被丢弃。因此，为了安全返回 &sum 指针，Go 必须从堆栈外为其分配内存。这就是堆的作⽤。

堆⽤于存储可能⽐创建它的函数存活时间更⻓的数据，以及任何使⽤指针在goroutine 之间共享数据。然⽽，这就提出了⼀个问题: 这些内存是如何被释放的。因为与堆栈分配不同，当创建堆分配的函数返回时，堆分配是不能被丢弃。

Go 使⽤其内置的垃圾收集器解决了这个问题。其实现的细节⾮常复杂，但是从宏观⻆度来看，它可以跟踪你的内存，如下图所示。在这⾥你可以看到三个goroutines，它们具有指向堆上绿⾊分配的指针。其中⼀些分配还有指向其他分配的指针，⽤绿⾊显示。此外，灰⾊分配可能指向绿⾊分配或相互指向，但它们本身并不被绿⾊分配引⽤。这些分配曾经是可到达的，但现在被认为是垃圾。如果在堆栈上分配它们的指针的函数返回，或者它们的值被覆盖，就会发⽣这种情况。GC 负责⾃动识别和释放这些分配。

![[Pasted image 20240325114208.png]]

执⾏ GC 涉及⼤量开销很⼤的图遍历和缓冲区的处理。它甚⾄需要定期 stop-the-world 阶段来停⽌整个程序执⾏。Go 现在的版本已经把 GC 的过程打到很恐怖的速度了(毫秒级)，剩余的⼤部分开销都是任何 GC 算法都跑不掉的的。事实上，Go 程序执⾏中 20-30% 的时间都开销在内存管理上并不罕⻅。

⼀般来说，GC 的成本与程序执⾏的堆分配量成正⽐。因此，在优化程序的内存开销时，需要注意的是:

- Reduce: 尝试将堆分配转换为堆栈分配，或者⼲脆避免它们。把堆上的指针数量打下来也会有帮助。
- Reuse: 复⽤堆分配，⽽不是⽤新的堆分配来替换它们。
- Recycle: 有些堆分配是⽆法避免的。让 GC ⾃动回收它们，并关注其他问
题。

与本指南中之前的⼼智模型⼀样，上⾯的流程都是对实际的情况做了简化。但是希望它能够很好地帮你理解本指南的其余部分，并激励你阅读更多关于这个主题的⽂章。推荐你必读的⼀篇⽂章[《Getting to Go: The Journey of Go'sGarbage Collector》]([Getting to Go: The Journey of Go's Garbage Collector - The Go Programming Language](https://go.dev/blog/ismmkeynote)) ，它让你很好地了解 Go 的 GC 多年来是如何进步的，以及它的改进速度。

## Go Profilers

以下是 Go runtime 内置 profilers 的概述。有关更多详细信息，请点击链接。

![[Pasted image 20240325114456.png]]

1. ⼀个 O(N) stop-the-world 的暂停，其中 N 是 goroutine 的数量。预计每个
goroutine 会有 ~1-10 微妙的暂停。
2. 完全坏了，不要尝试使⽤它。
3. 取决于 API 的情况。
#### CPU Profiler

Go 的 CPU profiler 可以帮助你确定代码的哪些部分占⽤了⼤量的 CPU 时间。

⚠请注意，CPU 时间跟我们体验的实际时间是不同的。例如，⼀个典型的 http请求可能需要 100ms 才能完成，但是在数据库上等待 95ms 时只消耗 5ms 的CPU 时间。⼀个请求也有可能 100ms ，但是如果两个 goroutine 并⾏地执⾏CPU 密集型⼯作，则需要花费 200ms 的 CPU。如果你对此感到困惑，请参阅 [[#Goroutine调度器]] 部分。

你可以通过各种 APIs 来控制 CPU profiler:

- go test -cpuprofile cpu.pprof 将运⾏你的测试并将 CPU profile 写⼊名为cpu.pprof 的⽂件。
-  [pprof.StartCPUProfile(w)]([pprof package - runtime/pprof - Go Packages](https://pkg.go.dev/runtime/pprof#StartCPUProfile)) 将 CPU profile 抓取到 w ，涵盖的时间跨度直到[pprof.StopCPUProfile()]([pprof package - runtime/pprof - Go Packages](https://pkg.go.dev/runtime/pprof#StopCPUProfile)) 被调⽤。
- [import _ "net/http/pprof"]([pprof package - net/http/pprof - Go Packages](https://pkg.go.dev/net/http/pprof)) 允许你通过点击默认 http 服务器的 GET /debug/pprof/profile?seconds=30 来请求 30s CPU profile，你可以通过 http.ListenAndServe("localhost:6060", nil) 来启动。
- [runtime.SetCPUProfileRate()]([runtime package - runtime - Go Packages](https://pkg.go.dev/runtime#SetCPUProfileRate)) 可以让你控制 CPU profile 的采样率。
- [runtime.SetCgoTraceback()]([runtime package - runtime - Go Packages](https://pkg.go.dev/runtime#SetCgoTraceback)) 可以⽤来获取 cgo 代码中的堆栈痕迹。[benesch/cgosymbolizer]([benesch/cgosymbolizer: cgosymbolizer teaches the Go runtime to include cgo frames in backtraces (github.com)](https://github.com/benesch/cgosymbolizer)) 有⼀个针对 Linux 和 macOS 的实现。

如果你需要⼀个可以⽴⻢看到效果的代码贴到 main() 函数⾥，你可以使⽤下⾯的代码:

```go
file, _ := os.Create("./cpu.pprof")
pprof.StartCPUProfile(file)
defer pprof.StopCPUProfile()
```

⽆论如何激活 CPU profiler，最终的 profile ⽂件本质上都是⼀个以⼆进制 [pprof]([go-profiler-notes/pprof.md at main · DataDog/go-profiler-notes (github.com)](https://github.com/DataDog/go-profiler-notes/blob/main/pprof.md))格式的堆栈跟踪表。这种表的简化版本如下:

![[Pasted image 20240325115147.png]]

CPU profiler 通过请求操作系统监视应⽤程序的 CPU 使⽤情况来捕获这些数据，并为每占⽤ 10ms 的 CPU 时间发送⼀个 SIGPROF 信号。操作系统还将内核代表应⽤程序所消耗的时间包括在这个监测中。由于信号传输速率取决于 CPU 的消耗，因此它是动态的，可以达到 N * 100Hz ，其中 N 是系统上逻辑 CPU 核⼼的数量。当 SIGPROF 信号到达时，Go 的信号处理程序捕获当前活动 goroutine 的堆栈跟踪，并在 profile 中增加相应的值。 cpu/nanoseconds 的值⽬前是直接从样本计数中推导出来的，因此它是冗余的，但很⽅便。

##### CPU Profiler 标签

Go 的 CPU profiler 的⼀个很吊的特性是你可以将任意键值对附加到 goroutine。这些标签将由从这个 goroutine 产⽣的任何 goroutine 继承，并显示在产⽣的profile ⽂件中。

让我们考虑下⾯的示例，它代表 user 执⾏⼀些 CPU work() 。通过使⽤[pprof.Labels()]([pprof package - runtime/pprof - Go Packages](https://pkg.go.dev/runtime/pprof#Labels))和pprof.Labels() API，我们可以将 user 与执⾏ work() 函数的goroutine 关联起来。此外，同⼀块代码中产⽣任何 goroutine 都会⾃动继承这些标签，例如 backgroundWork() goroutine。

```go
// +build ignore

package main

import (
	"context"
	"os"
	"runtime/pprof"
	"time"
)

func main() {
	pprof.StartCPUProfile(os.Stdout)
	defer pprof.StopCPUProfile()

	go work(context.Background(), "alice")
	go work(context.Background(), "bob")

	time.Sleep(50 * time.Millisecond)
}

func work(ctx context.Context, user string) {
	labels := pprof.Labels("user", user)
	pprof.Do(ctx, labels, func(_ context.Context) {
		go backgroundWork()
		directWork()
	})
}

func directWork() {
	for {
	}
}

func backgroundWork() {
	for {
	}
}
```

得到的 profile 将包括⼀个新的标签列，可能看起来像这样:

![[Pasted image 20240325115759.png]]

使⽤ pprof’s Graph 视图查看相同的档案也会包括以下标签:

![[Pasted image 20240325115822.png]]

如何使⽤这些标签取决于你。你可以包含⼀些东⻄⽐如 user ids 、 request ids、 http endpoints , subscription plan 或其他数据，这些东⻄可以让你更好地理解哪些类型的请求导致了⾼ CPU 利⽤率，即使它们是由相同的代码路径处理的。也就是说，使⽤标签会增加 pprof ⽂件的⼤⼩。因此，你可能应该先从低cardinality 标签（⽐如 endpoints）开始，⼀旦你确信它们不会影响你的应⽤程
序的性能，就应该转向⾼ cardinality 标签。

##### CPU利用率

由于 CPU profiler 的采样速率适应程序消耗的 CPU 数量，因此可以从 CPUprofile中获得 CPU 利⽤率。事实上，pprof 会⾃动为你做这件事。例如，下⾯的profile 取⾃⼀个平均 CPU 利⽤率为 147.77% 的程序:

```bash
 go tool pprof guide/cpu-utilization.pprof
Type: cpu
Time: Sep 9, 2021 at 11:34pm (CEST)
Duration: 1.12s, Total samples = 1.65s (147.77%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof)
```

另⼀种流⾏的表示 CPU 利⽤率的⽅法是 CPU 核⼼。在上⾯的例⼦中，程序在profiling 期间平均使⽤了 1.47 个 CPU 核⼼。

如果这个数值接近或⾼于 250% ，那么你不应该过于信任这个数值。但是，如果你看到的数字⾮常低，⽐如 10% ，这通常表明 CPU 消耗对你的应⽤程序来说是⼩ case。⼀个常⻅的错误是忽略这个数字，并开始担⼼某个特定的函数占⽤了相对于profile ⽂件其余部分的很⻓时间。当总体 CPU 利⽤率较低时，这通常是浪费时间，因为通过优化这个函数不会获得太多好处。
##### CPU Profiles的系统调用

如果你看到诸如 syscall.Read() 或 syscall.Write() 这样的系统调⽤在你的 CPUprofiles 中使⽤了⼤量的时间，请注意这只是内核中这些函数中占⽤的 CPU 时间。I/O 时间本身没有被跟踪。在系统调⽤上花费⼤量时间通常表明调⽤过多，因此增加缓冲区⼤⼩可能会有所帮助。对于这种更复杂的情况，你应该考虑使⽤Linux perf，因为它还可以向你显示内核堆栈跟踪，从⽽可能为你提供更多的线
索。

##### CPU Profiler局限

有⼀些已知的问题和 CPU profiler 的局限性，需要注意的是:

- 🐞 在 Linux 上的⼀个已知问题是 CPU profile 难以实现超过 250Hz 的采样率。这通常不是问题，但如果你的 CPU 利⽤率⾮常⾼，就会导致偏差。有关这⽅⾯的更多信息，可以看看 GitHub issue。同时你可以使⽤⽀持更⾼采样频率的 Linux perf。
- ⚠你可以在调⽤ runtime.SetCPUProfileRate() 之前调⽤ runtime.StartCPUProfile()来调整 CPU profile 的速率。这将打印⼀个警告： runtime: cannot set cpuprofile rate until previous profile has finished 。然⽽，它仍然在上述 bug的限制下⼯作。这个问题最初是在这⾥提出的，并且有⼀个被接受的改进 API的建议。
- ⚠⽬前，CPU profile 可以在堆栈跟踪中捕获的嵌套函数调⽤的最⼤数量是64。如果你的程序使⽤了⼤量的递归或其他导致调⽤函数堆栈的⽅法，你的CPU profile 将包括堆栈跟踪被截断。这意味着你将错过调⽤链中导致采样时处于活动状态的函数的部分。
#### Memory Profiler

Go memory(内存) profiler 可以帮助你识别代码中哪些部分执⾏了⼤量堆分配，以及在上⼀次垃圾收集期间有多少分配是仍可访问的。因此，memory profiler ⽣成的 profile 通常也称为 heap(堆) profile。

堆内存管理相关的活动通常占⽤ Go 进程消耗的 CPU 时间的 20-30% 。此外，由于减少了垃圾收集器扫描堆时发⽣的缓存抖动，⼲掉堆分配会产⽣⼆阶效应，从⽽加快代码的其他部分。这意味着优化内存分配通常⽐优化程序中与 cpu 绑定的代码路径更好使。

memory profiler不显示堆栈分配，因为这些分配通常⽐堆分开销⼩得多。有关详细信息，请参阅本指南的  [[#垃圾收集器]] 章节。

你可以通过各种 api 来控制 memory profiler:

- go test -memprofile mem.pprof 将运⾏你的测试并将 memory profile 写进mem.pprof 。
- [pprof.Lookup("allocs").WriteTo(w, 0)]([pprof package - runtime/pprof - Go Packages](https://pkg.go.dev/runtime/pprof#Lookup)) 将⼀个涵盖进程开始以来的时间的memory profile 写⼊到 w 。
- [import _ "net/http/pprof"]([pprof package - net/http/pprof - Go Packages](https://pkg.go.dev/net/http/pprof)) 允许你通过点击默认的 http 服务器 GET /debug/pprof/allocs?seconds=30 来请求 30 秒的 memory profile，你可以通过 http.ListenAndServe("localhost:6060", nil) 启动。这在内部也被称为delta profile 。
- [runtime.MemProfileRate]([runtime package - runtime - Go Packages](https://pkg.go.dev/runtime#MemProfileRate)) 允许你控制 memory profilee 的采样率。

如果你需要⼀个可以⽴⻢看到效果的代码贴到 main() 函数⾥，你可以使⽤下⾯的代码:

```go
file, _ := os.Create("./mem.pprof")
defer pprof.Lookup("allocs").WriteTo(file, 0)
defer runtime.GC()
```

⽆论如何激活 memory profiler，得到的 profile ⽂件本质上都是⼀个⼆进制pprof 格式的格式化的堆栈跟踪表。这种表的简化版本如下:

![[Pasted image 20240325133500.png]]

memory profile 包含两条主要信息:
- alloc_* : 程序从进程最开始以来所有的分配。
- insue_* : 程序从上次 GC 完到现在分配。

你可以将此信息⽤于各种⽤途。例如，你可以使⽤ alloc_* 数据来确定哪些代码路径可能会产⽣⼤量 GC 处理，并且随着时间的推移查看 inuse_* 数据可以帮助你调查程序的内存泄漏或内存使⽤率过⾼。

##### Allocs vs Heap Profile

[pprof.Lookup()]([pprof package - runtime/pprof - Go Packages](https://pkg.go.dev/runtime/pprof#Lookup)) 函数以及 net/http/pprof 包使⽤两个名称( allocs 和 heap )公开memory profile。两个 profile 都包含相同的数据，唯⼀的区别是 allocs profile将 alloc_space/bytes 设置为默认的样本类型，⽽ heap profile 默认设置为inuse_space/bytes 。pprof ⼯具使⽤它来决定默认情况下显示哪个示例类型。

##### Memory Profiler采样

为了保持⽐较低的开销，内存 profiler 使⽤泊松采样，因此平均每 512KiB 只有⼀次分配触发堆栈跟踪并将其添加到 profile 中。然⽽，在将 profile 写⼊最终的pprof ⽂件之前，runtime 将收集到的样本值除以抽样概率，从⽽对其进⾏扩展。

这意味着报告的分配数量应该是对实际分配数量的很好的估算。⽽不管你使⽤的是runtime.MemProfileRate。

对于⽣产中的分析，通常不必修改取样速率。这样做的唯⼀原因是，如果你担⼼在很少进⾏分配的情况下没有收集到⾜够的样本。

##### Memory Inuse vs RSS

⼀个常⻅的混淆是查看 inuse_space/bytes 样本类型的内存总量，并注意到它与操
作系统报告的 RSS 内存使⽤量不匹配。有多种原因可能导致:

- 根据定义，RSS 包括了很多不仅仅是 Go 堆内存的使⽤，例如 goroutine 堆栈、程序可执⾏⽂件、共享库以及 C 函数分配的内存所使⽤的内存。
- GC 可能不会⽴把空闲内存返回给操作系统，但在 Go 1.16 的 runtime 改回来了，这个就算⼩问题了。
- Go 使⽤ non-moving GC，因此在某些情况下，空闲堆内存可能会碎⽚化，从⽽阻⽌ Go 将其释放到操作系统。

##### Memory Profiler Implementation

下⾯的伪代码应该捕获 memory profiler 实现的基本东⻄，让你有个⼤概的印象。如下所示，Go runtime 中的 malloc() 函数使⽤ poisson_sample(size) 来确定是否应该对分配进⾏取样。如果是，则以堆栈跟踪 s 作为 mem_profile 哈希表中的 key，⽤来增加 allocs 和 alloc_bytes 计数器。此外，
track_profiled(object, s) 调⽤将 object 标记为堆上的采样分配，并将堆栈跟踪s 与其关联。

```go
func malloc(size):
	object = ... // allocation magic
	if poisson_sample(size):
		s = stacktrace()
		mem_profile[s].allocs++
		mem_profile[s].alloc_bytes += size
		track_profiled(object, s)
return object
```

当 GC 确定是时候释放分配的对象时，它调⽤ sweep() ，使⽤is_profiled(object) 来检查 object 是否被标记为采样对象。如果是，它将检索导致分配的堆栈跟踪 s ，并在 mem_profile 内为它增加 frees 和 free_bytes 计数器。

```go
func sweep(object):
	if is_profiled(object)
		s = alloc_stacktrace(object)
		mem_profile[s].frees++
		mem_profile[s].free_bytes += sizeof(object)
		// deallocation magic
```

free_* 计数器本身不包含在最终 memory profile 中。相反，它们通过简单的allocs - frees 减法计算 profile 中的 insue_* 计数器。另外，最终的输出值是通过取样概率除以它们的⽐例⽽得到的。

##### Memory Profiler的局限

memory profiler 有⼀些已知的问题还有局限性，需要注意的是:

- runtime.MemProfileRate 必须尽可能早地在程序启动时修改⼀次，例如在main() 的开头。写⼊这个值本质上是⼀个产⽣的数据竞争很⼩，在程序执⾏期间多次更改它会产⽣不正确的配置⽂件
- ⚠ 在调试潜在的内存泄漏时，memory profiler 可以显示这些分配的是哪⾥创建的，但它⽆法告诉你哪些指针还在保持引⽤。这么些年，⼀直有⼈想解决掉这个问题，但没有⼀个适⽤于最新版本的 Go
- ⚠ CPU Profiler Labels 或者其他类似的东⻄不受 memory profiler ⽀持。在⽬前的实现中很难添加这个功能，因为它可能会在保存 memory profiler 数据的内部哈希映射中造成内存泄漏。
- cgo C 代码所做的分配不会显示在 memory profile ⽂件中。
- Memory profile 可能是两个垃圾收集周期前的数据。如果你想要⼀个更⼀致的时间点快照，可以考虑在请求内存配置⽂件之前调⽤ runtime.GC() net/http/pprof 接受 ?gc=1 的参数。更多信息请参阅 runtime.MemProfile()⽂档， 以及 mprof.go 中关于 memRecord 的注释。
- memory profiler 可以在堆栈跟踪中捕获的嵌套函数调⽤的最⼤数量⽬前是32 , 有关超过此限制时会发⽣什么情况的更多信息，请参阅 CPU Profiler Limitations。
- 保存 memory profile ⽂件的内部哈希表没有⼤⼩限制。这意味着它的⼤⼩会不断增⻓，直到它涵盖您的代码库中的所有分配代码路径。这在实践中不是问题，但如果您正在观察进程的内存使⽤情况，它可能看起来像⼀个⽐较⼩的内存泄漏

#### ThreadCreate Profiler

Threadcreate profile 旨在显示导致创建新 OS 线程的堆栈跟踪。然⽽，它从2013 年就已经坏了，所以⼤家记得不要⽤这个