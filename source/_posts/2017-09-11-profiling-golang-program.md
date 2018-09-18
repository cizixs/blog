---
layout: post
title: "使用 pprof 和火焰图调试 golang 应用"
excerpt: "在计算机性能调试领域里，profiling 就是对应用的画像，这里画像就是应用使用 CPU 和内存的情况。也就是说应用使用了多少 CPU 资源？都是哪些部分在使用？每个函数使用的比例是多少？有哪些函数在等待 CPU 资源？知道了这些，我们就能对应用进行规划，也能快速定位性能瓶颈。"
categories: blog
tags: [golang, pprof, flamegraph]
comments: true
share: true
---

## 什么是 Profiling?

Profiling 这个词比较难翻译，一般译成`画像`。比如在案件侦破的时候会对嫌疑人做画像，从犯罪现场的种种证据，找到嫌疑人的各种特征，方便对嫌疑人进行排查；还有就是互联网公司会对用户信息做画像，通过了解用户各个属性（年龄、性别、消费能力等），方便为用户推荐内容或者广告。

在计算机性能调试领域里，profiling 就是对应用的画像，这里画像就是应用使用 CPU 和内存的情况。也就是说应用使用了多少 CPU 资源？都是哪些部分在使用？每个函数使用的比例是多少？有哪些函数在等待 CPU 资源？知道了这些，我们就能对应用进行规划，也能快速定位性能瓶颈。

golang 是一个对性能特别看重的语言，因此语言中自带了 profiling 的库，这篇文章就要讲解怎么在 golang 中做 profiling。

在 go 语言中，主要关注的应用运行情况主要包括以下几种：

- CPU profile：报告程序的 CPU 使用情况，按照一定频率去采集应用程序在 CPU 和寄存器上面的数据
- Memory Profile（Heap Profile）：报告程序的内存使用情况
- Block Profiling：报告 goroutines 不在运行状态的情况，可以用来分析和查找死锁等性能瓶颈
- Goroutine Profiling：报告 goroutines 的使用情况，有哪些 goroutine，它们的调用关系是怎样的

## 两种收集方式

做 Profiling 第一步就是怎么获取应用程序的运行情况数据。go 语言提供了 `runtime/pprof` 和 `net/http/pprof` 两个库，这部分我们讲讲它们的用法以及使用场景。

### 工具型应用

如果你的应用是一次性的，运行一段时间就结束。那么最好的办法，就是在应用退出的时候把 profiling 的报告保存到文件中，进行分析。对于这种情况，可以使用 [`runtime/pprof` 库](https://golang.org/pkg/runtime/pprof/)。

`pprof` 封装了很好的接口供我们使用，比如要想进行 CPU Profiling，可以调用 `pprof.StartCPUProfile()` 方法，它会对当前应用程序进行 CPU profiling，并写入到提供的参数中（`w io.Writer`），要停止调用 `StopCPUProfile()` 即可。

去除错误处理只需要三行内容，一般把部分内容写在 `main.go` 文件中，应用程序启动之后就开始执行：

```
f, err := os.Create(*cpuprofile)
...
pprof.StartCPUProfile(f)
defer pprof.StopCPUProfile()
```

应用执行结束后，就会生成一个文件，保存了我们的 CPU profiling 数据。

想要获得内存的数据，直接使用 `WriteHeapProfile` 就行，不用 `start` 和 `stop` 这两个步骤了：

```
f, err := os.Create(*memprofile)
pprof.WriteHeapProfile(f)
f.Close()
```

### 服务型应用

如果你的应用是一直运行的，比如 web 应用，那么可以使用 `net/http/pprof` 库，它能够在提供 HTTP 服务进行分析。

如果使用了默认的 `http.DefaultServeMux`（通常是代码直接使用 `http.ListenAndServe("0.0.0.0:8000", nil)`），只需要添加一行：

```
import _ "net/http/pprof"
```

如果你使用自定义的 `Mux`，则需要手动注册一些路由规则：

```
r.HandleFunc("/debug/pprof/", pprof.Index)
r.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
r.HandleFunc("/debug/pprof/profile", pprof.Profile)
r.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
r.HandleFunc("/debug/pprof/trace", pprof.Trace)
```

不管哪种方式，你的 HTTP 服务都会多出 `/debug/pprof` endpoint，访问它会得到类似下面的内容：

```
/debug/pprof/

profiles:
0	block
62	goroutine
444	heap
30	threadcreate

full goroutine stack dump
```

这个路径下还有几个子页面：

- `/debug/pprof/profile`：访问这个链接会自动进行 CPU profiling，持续 30s，并生成一个文件供下载
- `/debug/pprof/heap`： Memory Profiling 的路径，访问这个链接会得到一个内存 Profiling 结果的文件
- `/debug/pprof/block`：block Profiling 的路径
- `/debug/pprof/goroutines`：运行的 goroutines 列表，以及调用关系

## go tool pprof 命令：获取和分析 Profiling 数据

能通过对应的库获取想要的 Profiling 数据之后（不管是文件还是 http），下一步就是要对这些数据进行保存和分析，我们可以使用 `go tool pprof` 命令行工具。

在后面我们会生成调用关系图和火焰图，需要安装 `graphviz` 软件包，在 ubuntu 系统可以使用下面的命令：

```
$ sudo apt-get install -y graphviz
```

**NOTE**：获取的 Profiling 数据是动态的，要想获得有效的数据，请保证应用处于较大的负载（比如正在生成中运行的服务，或者通过其他工具模拟访问压力）。否则如果应用处于空闲状态，得到的结果可能没有任何意义。

### CPU Profiling

`go tool pprof` 最简单的使用方式为 `go tool pprof [binary] [source]`，`binary` 是应用的二进制文件，用来解析各种符号；`source` 表示 profile 数据的来源，可以是本地的文件，也可以是 http 地址。比如：

```
➜  go tool pprof ./hyperkube http://172.16.3.232:10251/debug/pprof/profile
Fetching profile from http://172.16.3.232:10251/debug/pprof/profile
Please wait... (30s)
Saved profile in /home/cizixs/pprof/pprof.hyperkube.172.16.3.232:10251.samples.cpu.002.pb.gz
Entering interactive mode (type "help" for commands)
(pprof) 
```

这个命令会进行 CPU profiling 分析，等待一段时间（默认是 30s，如果在 url 最后加上 `?seconds=60` 参数可以调整采集数据的时间为 60s）之后，我们就进入了一个交互式命令行，可以对解析的结果进行查看和导出。可以通过 `help` 来查看支持的自命令有哪些。

一个有用的命令是 `topN`，它列出最耗时间的地方：

```
(pprof) top10
130ms of 360ms total (36.11%)
Showing top 10 nodes out of 180 (cum >= 10ms)
      flat  flat%   sum%        cum   cum%
      20ms  5.56%  5.56%      100ms 27.78%  encoding/json.(*decodeState).object
      20ms  5.56% 11.11%       20ms  5.56%  runtime.(*mspan).refillAllocCache
      20ms  5.56% 16.67%       20ms  5.56%  runtime.futex
      10ms  2.78% 19.44%       10ms  2.78%  encoding/json.(*decodeState).literalStore
      10ms  2.78% 22.22%       10ms  2.78%  encoding/json.(*decodeState).scanWhile
      10ms  2.78% 25.00%       40ms 11.11%  encoding/json.checkValid
      10ms  2.78% 27.78%       10ms  2.78%  encoding/json.simpleLetterEqualFold
      10ms  2.78% 30.56%       10ms  2.78%  encoding/json.stateBeginValue
      10ms  2.78% 33.33%       10ms  2.78%  encoding/json.stateEndValue
      10ms  2.78% 36.11%       10ms  2.78%  encoding/json.stateInString
```

每一行表示一个函数的信息。前两列表示函数在 CPU 上运行的时间以及百分比；第三列是当前所有函数累加使用 CPU 的比例；第四列和第五列代表这个函数以及子函数运行所占用的时间和比例（也被称为`累加值 cumulative`），应该大于等于前两列的值；最后一列就是函数的名字。如果应用程序有性能问题，上面这些信息应该能告诉我们时间都花费在哪些函数的执行上了。

pprof 不仅能打印出最耗时的地方(`top`)，还能列出函数代码以及对应的取样数据(`list`)、汇编代码以及对应的取样数据(`disasm`)，而且能以各种样式进行输出，比如 svg、gv、callgrind、png、gif等等。

其中一个非常便利的是 `web` 命令，在交互模式下输入 `web`，就能自动生成一个 `svg` 文件，并跳转到浏览器打开，生成了一个函数调用图：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1fjfq8wjqunj31fa0q3afd.jpg)

这个调用图包含了更多的信息，而且可视化的图像能让我们更清楚地理解整个应用程序的全貌。图中每个方框对应一个函数，方框越大代表执行的时间越久（包括它调用的子函数执行时间，但并不是正比的关系）；方框之间的箭头代表着调用关系，箭头上的数字代表被调用函数的执行时间。

因为原图比较大，这里只截取了其中一部分，但是能明显看到 `encoding/json.(*decodeState).object` 是这里耗时比较多的地方，而且能看到它调用了哪些函数，分别函数多少。这些更详细的信息对于定位和调优性能是非常有帮助的！

要想更细致分析，就要精确到代码级别了，看看每行代码的耗时，直接定位到出现性能问题的那行代码。`pprof` 也能做到，`list` 命令后面跟着一个正则表达式，就能查看匹配函数的代码以及每行代码的耗时：

```
(pprof) list podFitsOnNode
Total: 120ms
ROUTINE ======================== k8s.io/kubernetes/plugin/pkg/scheduler.podFitsOnNode in /home/cizixs/go/src/k8s.io/kubernetes/_output/local/go/src/k8s.io/kubernetes/plugin/pkg/scheduler/generic_scheduler.go
         0       20ms (flat, cum) 16.67% of Total
         .          .    230:
         .          .    231:// Checks whether node with a given name and NodeInfo satisfies all predicateFuncs.
         .          .    232:func podFitsOnNode(pod *api.Pod, meta interface{}, info *schedulercache.NodeInfo, predicateFuncs map[string]algorithm.FitPredicate) (bool, []algorithm.PredicateFailureReason, error) {
         .          .    233:	var failedPredicates []algorithm.PredicateFailureReason
         .          .    234:	for _, predicate := range predicateFuncs {
         .       20ms    235:		fit, reasons, err := predicate(pod, meta, info)
         .          .    236:		if err != nil {
         .          .    237:			err := fmt.Errorf("SchedulerPredicates failed due to %v, which is unexpected.", err)
         .          .    238:			return false, []algorithm.PredicateFailureReason{}, err
         .          .    239:		}
         .          .    240:		if !fit {
```

如果想要了解对应的汇编代码，可以使用 `disadm <regex>` 命令。这两个命令虽然强大，但是在命令行中查看代码并不是很方面，所以你可以使用 `weblist` 命令，用法和两者一样，但它会在浏览器打开一个页面，能够同时显示源代码和汇编代码。 

**NOTE**：更详细的 pprof 使用方法可以参考 `pprof --help` 或者 [pprof 文档](https://github.com/google/pprof/blob/master/doc/pprof.md)。

### Memory Profiling

要想获得内存使用 Profiling 信息，只需要把数据源修改一下就行（对于 http 方式来说就是修改 url 的地址，从 `/debug/pprof/profile` 改成 `/debug/pprof/heap`）：

```
➜  go tool pprof ./hyperkube http://172.16.3.232:10251/debug/pprof/heap        
Fetching profile from http://172.16.3.232:10251/debug/pprof/heap
Saved profile in /home/cizixs/pprof/pprof.hyperkube.172.16.3.232:10251.inuse_objects.inuse_space.002.pb.gz
Entering interactive mode (type "help" for commands)
(pprof)
```

和 CPU Profiling 使用一样，使用 `top N` 可以打印出使用内存最多的函数列表：

```
(pprof) top
11712.11kB of 14785.10kB total (79.22%)
Dropped 580 nodes (cum <= 73.92kB)
Showing top 10 nodes out of 146 (cum >= 512.31kB)
      flat  flat%   sum%        cum   cum%
 2072.09kB 14.01% 14.01%  2072.09kB 14.01%  k8s.io/kubernetes/vendor/github.com/beorn7/perks/quantile.NewTargeted
 2049.25kB 13.86% 27.87%  2049.25kB 13.86%  k8s.io/kubernetes/pkg/api/v1.(*ResourceRequirements).Unmarshal
 1572.28kB 10.63% 38.51%  1572.28kB 10.63%  k8s.io/kubernetes/vendor/github.com/beorn7/perks/quantile.(*stream).merge
 1571.34kB 10.63% 49.14%  1571.34kB 10.63%  regexp.(*bitState).reset
 1184.27kB  8.01% 57.15%  1184.27kB  8.01%  bytes.makeSlice
 1024.16kB  6.93% 64.07%  1024.16kB  6.93%  k8s.io/kubernetes/pkg/api/v1.(*ObjectMeta).Unmarshal
  613.99kB  4.15% 68.23%  2150.63kB 14.55%  k8s.io/kubernetes/pkg/api/v1.(*PersistentVolumeClaimList).Unmarshal
  591.75kB  4.00% 72.23%  1103.79kB  7.47%  reflect.Value.call
  520.67kB  3.52% 75.75%   520.67kB  3.52%  k8s.io/kubernetes/vendor/github.com/gogo/protobuf/proto.RegisterType
  512.31kB  3.47% 79.22%   512.31kB  3.47%  k8s.io/kubernetes/pkg/api/v1.(*PersistentVolumeClaimStatus).Unmarshal
```

每一列的含义也是类似的，只不过从 CPU 使用时间变成了内存使用大小，就不多解释了。

类似的，`web` 命令也能生成 `svg` 图片在浏览器中打开，从中可以看到函数调用关系，以及每个函数的内存使用多少。

需要注意的是，默认情况下，统计的是内存使用大小，如果执行命令的时候加上 `--inuse_objects` 可以查看每个函数分配的对象数；`--alloc-space` 查看分配的内存空间大小。

这里还要提两个比较有用的方法，如果应用比较复杂，生成的调用图特别大，看起来很乱，有两个办法可以优化：

- 使用 `web funcName` 的方式，只打印和某个函数相关的内容
- 运行 `go tool pprof` 命令时加上 `--nodefration=0.05` 参数，表示如果调用的子函数使用的 CPU、memory 不超过 5%，就忽略它，不要显示在图片中

pprof 已经支持动态的 web 浏览方式：https://github.com/google/pprof/commit/f83a3d89c18c445178f794d525bf3013ef7b3330

## go-torch 和火焰图

火焰图（Flame Graph）是 Bredan Gregg 创建的一种性能分析图表，因为它的样子近似 🔥而得名。上面的 profiling 结果也转换成火焰图，如果对火焰图比较了解可以手动来操作，不过这里我们要介绍一个工具：[go-torch](https://github.com/uber/go-torch)。这是 uber 开源的一个工具，可以直接读取 golang profiling 数据，并生成一个火焰图的 svg 文件。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1fjc5nh6x52j30xc0litck.jpg)

火焰图 svg 文件可以通过浏览器打开，它对于调用图的最优点是它是动态的：可以通过点击每个方块来 zoom in 分析它上面的内容。

火焰图的调用顺序从下到上，每个方块代表一个函数，它上面一层表示这个函数会调用哪些函数，方块的大小代表了占用 CPU 使用的长短。火焰图的配色并没有特殊的意义，默认的红、黄配色是为了更像火焰而已。

go-torch 工具的使用非常简单，没有任何参数的话，它会尝试从 `http://localhost:8080/debug/pprof/profile` 获取 profiling 数据。它有三个常用的参数可以调整：

- `-u --url`：要访问的 URL，这里只是主机和端口部分
- `-s --suffix`：pprof profile 的路径，默认为 `/debug/pprof/profile`
- `--seconds`：要执行 profiling 的时间长度，默认为 30s

要生成火焰图，需要事先安装 [FlameGraph](https://github.com/brendangregg/FlameGraph)工具，这个工具的安装很简单，只要把对应的可执行文件放到 `$PATH` 目录下就行。

## 和测试工具的集成

go test 命令有两个参数和 pprof 相关，它们分别指定生成的 CPU 和 Memory profiling 保存的文件：

- `-cpuprofile`：cpu profiling 数据要保存的文件地址
- `-memprofile`：memory profiling 数据要报文的文件地址

比如下面执行测试的同时，也会执行 CPU profiling，并把结果保存在 `cpu.prof` 文件中：

```
$ go test -bench . -cpuprofile=cpu.prof
```

执行结束之后，就会生成 `main.test` 和 `cpu.prof`  文件。要想使用 `go tool pprof`，需要指定的二进制文件就是 `main.test`。

需要注意的是，Profiling 一般和性能测试一起使用，这个原因在前文也提到过，只有应用在负载高的情况下 Profiling 才有意义。

## 参考资料

- [The Go Blog: Profiling Go Programs](https://blog.golang.org/profiling-go-programs)
- [go command tutorial: go tool pprof   ](https://github.com/hyper0x/go_command_tutorial/blob/master/0.12.md)
- [Profiling and optimizing Go web applications](http://artem.krylysov.com/blog/2017/03/13/profiling-and-optimizing-go-web-applications/)
- [Debugging performance issues in Go programs](https://software.intel.com/en-us/blogs/2014/05/10/debugging-performance-issues-in-go-programs)