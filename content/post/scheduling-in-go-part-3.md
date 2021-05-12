---
title: "Go语言调度（三）：并发"
date: 2021-05-12T18:33:42+08:00
draft: false
---

## 序言
这是一个由三部分组成的系列文章，翻译而来，[原文出处请戳](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)。 将提供对Go语言调度器背后的机制和语义的理解。这是第三篇，这篇文章将着重于并发。

## 简介
当我在解决一个问题，特别是新问题的时候，我并不会一开始就考虑到并发是否是一个好的选择。我最开始会用串行的方式并保证能够正常工作。然后经过可读性和技术评审，我会开始思考如果用并发的方式是否合理且可实践的。有些时候并发毫无疑问是一个好的选择，有的时候却又不太确定。

在第一篇文章中，我解释了OS调度器的机制和语义，如果你打算写多线程程序，我相信这是很重要的。在第二部分，我解释了GO的语义，我相信这对于理解如何写GO的并发程序是很重要的。在这篇文章中，我将把GO和OS调度器的机制和语义结合在一起，来提供什么是并发什么不是的更深的理解。

本篇文章目的：
* 提供关于确定工作负载是否适合使用并发性时必须考虑的语义的指导。
* 向您展示不同类型的工作负载如何改变语义，从而改变您想要做出的工程决策。

## 什么是并发
并发意味着“无序”执行。接受一组指令，本来是顺序执行的，然后找到一种方法，使其无序执行并产生和顺序执行相同的结果。对于你面前的问题，无序执行显然能创造价值。我说的价值，是指增加提升性能但同时会增加复杂度。根据你的问题，无序执行也或许是不可能的或者没有价值。

理解并发不等于并行也是重要的。并行意味着同时执行2个或更多的指令。这是一个和并发不同的概念。当你有至少两个操作系统和硬件线程和至少两个Goroutines，每个goroutine在独立的操作系统或硬件上执行指令，这才可能是并行的。

####  Figure 1
![96_figure1.png](https://upload-images.jianshu.io/upload_images/13462240-d9d37aff7fdd4a44.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在图1中，两个逻辑处理器（P）和他们独立的操作系统线程（M）绑定在机器中独立的硬件线程上（Core）。你能看到两个Goroutines（G1和G2）在并行执行， 同时在不同的硬件操作系统上执行指令。在每个逻辑处理器中，3个Goroutines在轮流的使用各自的操作系统线程。所有的Goroutines都在并发的执行，无序的执行指令和共享操作系统线程。

## 工作负载
你如何知道什么时候无序执行是可行并且有效的？理解你的问题需要处理哪种工作负载是一个好的出发点。当考虑并发时，理解以下两种工作负载是很重要的。
* CPU密集型（CPU-Bound）：这种工作负载无法创造使Goroutines移入或移出等待状态（waiting states）的场景，因为工作负载一直在做计算。计算Pi小数点后的第N位就是这样一种场景
* IO密集型（IO-Bound）：这种工作负载将会使Goroutines自然地进入等待状态（waiting states）。这些工作包括请求通过网络访问资源，或对操作系统进行系统调用，或等待事件发生。Goroutines需要读取文件也是IO密集型的，我也会把导致Goroutines等待的同步事件（mutexes，atomic）也归类为这种类型。

对于CPU密集型的工作负载，你需要并行来提升并发。因为这种类型的工作负载，goroutine不会自然的从等待状态中移入或移出，所以单个操作系统/硬件线程处理多个Goroutines是低效的。拥有比操作系统/硬件线程更多的Goroutines会降低工作负载的执行速度，因为在操作系统线程中移动Goroutines延迟成本(时间)比较高。下文切换为你的工作负载创建一个“Stop The World”事件，因为在切换期间没有任何工作负载被执行(原本上可能会执行)。

对于IO密集型的工作负载，你不用提高并行来增加并发。因为Goroutines天然的移入/移出等待状态，所以一个单独的操作系统/硬件线程就能高效地处理多个Goroutines。比操作系统/硬件线程更多的Goroutines能够加速任务执行，因为从系统线程上移动Goroutines的延迟花费并不会创建一个“Stop The World”事件。你的工作负载自然的停止了，然后允许其他的Goroutines高效地利用同一个操作系统/硬件线程，而不是让他空闲。

如何知道每个硬件线程多少个Goroutines能提供最好的吞吐量呢？太少的Goroutines将会导致更多的闲置时间，太多的Goroutines将会有更多的上下文切换延迟时间。这些都是你需要考虑的问题，但是不在本篇的讨论范围中。

现在重要的是看一些代码来巩固你鉴别工作负载是否能提升并发，如果不能的话是否有必要提高并行的能力

## 加数

我们不必用复杂的代码来具象和理解这些语义。请看下面这个累加一系列整数的方法：add。

#### Listing 1
https://play.golang.org/p/r9LdqUsEzEz

```
36 func add(numbers []int) int {
37     var v int
38     for _, n := range numbers {
39         v += n
40     }
41     return v
42 }
```

在这段代码中，定义的add方法接受一个整数数组，返回数组的和。

问题：方法add是适合于无序执行的工作负载吗？我相信答案是肯定的。整数集合可以被分为更小的可以并发执行的列表。一旦被拆分的列表分别被求和,累加值列表就可以全部再被加起来，得到和顺序执行一样的结果。

然而这将产生另外一个问题，应该分成多少更小的独立列表以获得最佳的吞吐量呢？为了回答这个问题，你应该知道add是什么类型的工作负载。方法add是一个cpu密集型的工作负载，因为算法不做任何其他会导致goroutine进入等待状态的事情，而只做数学计算。这意味着只需要为每个操作系统/硬件线程分配一个goroutine就能够获得最佳的吞吐量。

列表2是并发版本的方法add。

注：有很多种方法来写add的并发版本。不用纠结于我实现的这一版，如果你有更可读的并且性能一样或更好的版本，我很愿意你能够分享他。

#### Listing 2
https://play.golang.org/p/r9LdqUsEzEz

```
44 func addConcurrent(goroutines int, numbers []int) int {
45     var v int64
46     totalNumbers := len(numbers)
47     lastGoroutine := goroutines - 1
48     stride := totalNumbers / goroutines
49
50     var wg sync.WaitGroup
51     wg.Add(goroutines)
52
53     for g := 0; g < goroutines; g++ {
54         go func(g int) {
55             start := g * stride
56             end := start + stride
57             if g == lastGoroutine {
58                 end = totalNumbers
59             }
60
61             var lv int
62             for _, n := range numbers[start:end] {
63                 lv += n
64             }
65
66             atomic.AddInt64(&v, int64(lv))
67             wg.Done()
68         }(g)
69     }
70
71     wg.Wait()
72
73     return int(v)
74 }
```

并发版本明显比串行版本更复杂，增加的复杂度是值得的吗？回答这个问题最好的方法是做基准测试。

#### Listing 3

```
func BenchmarkSequential(b *testing.B) {
    for i := 0; i < b.N; i++ {
        add(numbers)
    }
}

func BenchmarkConcurrent(b *testing.B) {
    for i := 0; i < b.N; i++ {
        addConcurrent(runtime.NumCPU(), numbers)
    }
}
```

列表3中是基准测试方法。下面是只有一个操作系统/硬件线程对所有Goroutines可用时的结果。串行的版本只使用了一个Goroutines，而并发的版本使用了runtime.NumCPU个，在我的机器上也就是8个Goroutines.这种情况下，并发的版本并没有利用并行来提高并发。

#### Listing 4

```
10 Million Numbers using 8 goroutines with 1 core
2.9 GHz Intel 4 Core i7
Concurrency WITHOUT Parallelism
-----------------------------------------------------------------------------
$ GOGC=off go test -cpu 1 -run none -bench . -benchtime 3s
goos: darwin
goarch: amd64
pkg: github.com/ardanlabs/gotraining/topics/go/testing/benchmarks/cpu-bound
BenchmarkSequential      	    1000	   5720764 ns/op : ~10% Faster
BenchmarkConcurrent      	    1000	   6387344 ns/op
BenchmarkSequentialAgain 	    1000	   5614666 ns/op : ~13% Faster
BenchmarkConcurrentAgain 	    1000	   6482612 ns/op
```

注：在你的本地环境跑基准测试是很复杂的。有很多的因素会导致你的benchmark不准确。请确保你的机器尽可能空闲并且多跑几次测试。你要确保你看到的结果是一致的，通过测试工具运行两次基准测试可以获得最一致的结果。

列表4中的基准测试表明在只有一个操作系统/硬件线程情况下，串行版本会比并行版本快大概10%～13%。这正是我所期望的，因为并发版本在单个OS线程上有上下文切换的开销和对Goroutines的管理。

下面的结果是每个Goroutines都能有一个独立的操作系统/硬件线程。串行的版本只有一个Goroutines，并行的版本则使用runtime.NumCPU个Goroutines，在我的机器上，这个数量是8.在这种情况下，并发的版本可以利用并行来提高并发。

#### Listing 5

```
10 Million Numbers using 8 goroutines with 8 cores
2.9 GHz Intel 4 Core i7
Concurrency WITH Parallelism
-----------------------------------------------------------------------------
$ GOGC=off go test -cpu 8 -run none -bench . -benchtime 3s
goos: darwin
goarch: amd64
pkg: github.com/ardanlabs/gotraining/topics/go/testing/benchmarks/cpu-bound
BenchmarkSequential-8        	    1000	   5910799 ns/op
BenchmarkConcurrent-8        	    2000	   3362643 ns/op : ~43% Faster
BenchmarkSequentialAgain-8   	    1000	   5933444 ns/op
BenchmarkConcurrentAgain-8   	    2000	   3477253 ns/op : ~41% Faster
```

列表5中的基准测试结果表明， 在使用多个操作系统/硬件线程的情况下，并发版本比串行版本快了41%～43%。这也正是我所期望的，因为Goroutines现在在并行执行。8个Goroutines在同时地执行指令。

## 排序

理解并不是所有CPU密集型的工作负载都适合并发是很重要的。特别是当分开进行工作或/和合并所有结果都很昂贵的场景。一个例子就是冒泡排序算法。请看下面冒泡排序算法的实现。


#### Listing 6
https://play.golang.org/p/S0Us1wYBqG6

```
01 package main
02
03 import "fmt"
04
05 func bubbleSort(numbers []int) {
06     n := len(numbers)
07     for i := 0; i < n; i++ {
08         if !sweep(numbers, i) {
09             return
10         }
11     }
12 }
13
14 func sweep(numbers []int, currentPass int) bool {
15     var idx int
16     idxNext := idx + 1
17     n := len(numbers)
18     var swap bool
19
20     for idxNext < (n - currentPass) {
21         a := numbers[idx]
22         b := numbers[idxNext]
23         if a > b {
24             numbers[idx] = b
25             numbers[idxNext] = a
26             swap = true
27         }
28         idx++
29         idxNext = idx + 1
30     }
31     return swap
32 }
33
34 func main() {
35     org := []int{1, 3, 2, 4, 8, 6, 7, 2, 3, 0}
36     fmt.Println(org)
37
38     bubbleSort(org)
39     fmt.Println(org)
40 }
```

问题： 方法bubbleSort是适合无序执行的工作负载吗？我相信答案是No。整数集合可以被分成更小的整数集合，然后并发的分开排序。然后当所有并发工作完成后，没有方法能够高效地把排序结果再排序在一起。下面是一个并发冒泡排序的例子。

#### Listing 8

```
01 func bubbleSortConcurrent(goroutines int, numbers []int) {
02     totalNumbers := len(numbers)
03     lastGoroutine := goroutines - 1
04     stride := totalNumbers / goroutines
05
06     var wg sync.WaitGroup
07     wg.Add(goroutines)
08
09     for g := 0; g < goroutines; g++ {
10         go func(g int) {
11             start := g * stride
12             end := start + stride
13             if g == lastGoroutine {
14                 end = totalNumbers
15             }
16
17             bubbleSort(numbers[start:end])
18             wg.Done()
19         }(g)
20     }
21
22     wg.Wait()
23
24     // Ugh, we have to sort the entire list again.
25     bubbleSort(numbers)
26 }
```

#### Listing 9

```
Before:
  25 51 15 57 87 10 10 85 90 32 98 53
  91 82 84 97 67 37 71 94 26  2 81 79
  66 70 93 86 19 81 52 75 85 10 87 49

After:
  10 10 15 25 32 51 53 57 85 87 90 98
   2 26 37 67 71 79 81 82 84 91 94 97
  10 19 49 52 66 70 75 81 85 86 87 93
```

因为冒泡排序的本质是遍历整个数组，bubbleSort的第25行会否定所有使用并发带来的已排序好的部分。所以使用并发并不会对冒泡排序带来性能上的提升。

## 读取文件

我们已经看过两种CPU密集型的工作负载了，那IO密集型的呢？这些Goroutines从等待状态切换的时候语义是一样的吗？请看下面的读取文件和文件搜索的IO密集型的工作负载。

第一个版本是串行版本，方法名称叫find

#### Listing 10
https://play.golang.org/p/8gFe5F8zweN

```
42 func find(topic string, docs []string) int {
43     var found int
44     for _, doc := range docs {
45         items, err := read(doc)
46         if err != nil {
47             continue
48         }
49         for _, item := range items {
50             if strings.Contains(item.Description, topic) {
51                 found++
52             }
53         }
54     }
55     return found
56 }
```

下面是被find调用的方法read的实现。

#### Listing 11
https://play.golang.org/p/8gFe5F8zweN

```
33 func read(doc string) ([]item, error) {
34     time.Sleep(time.Millisecond) // Simulate blocking disk read.
35     var d document
36     if err := xml.Unmarshal([]byte(file), &d); err != nil {
37         return nil, err
38     }
39     return d.Channel.Items, nil
40 }
```

列表11中的read方法调用了time.Sleep,这个方法模拟了从硬盘读文件的系统调用的延迟。延迟的一致性对准确测量并发版本和串行版本的性能是非常重要的。

有了串行版本后，下面是并发版本。

#### Listing 12
https://play.golang.org/p/8gFe5F8zweN

```
58 func findConcurrent(goroutines int, topic string, docs []string) int {
59     var found int64
60
61     ch := make(chan string, len(docs))
62     for _, doc := range docs {
63         ch <- doc
64     }
65     close(ch)
66
67     var wg sync.WaitGroup
68     wg.Add(goroutines)
69
70     for g := 0; g < goroutines; g++ {
71         go func() {
72             var lFound int64
73             for doc := range ch {
74                 items, err := read(doc)
75                 if err != nil {
76                     continue
77                 }
78                 for _, item := range items {
79                     if strings.Contains(item.Description, topic) {
80                         lFound++
81                     }
82                 }
83             }
84             atomic.AddInt64(&found, lFound)
85             wg.Done()
86         }()
87     }
88
89     wg.Wait()
90
91     return int(found)
92 }
```

并发版本明显比串行版本更复杂，但这是值得的吗？回到这个问题最好的方式就是进行基准测试。对于这些基准测试，我使用了1000个文档，并关闭了垃圾回收。

#### Listing 13

```
func BenchmarkSequential(b *testing.B) {
    for i := 0; i < b.N; i++ {
        find("test", docs)
    }
}

func BenchmarkConcurrent(b *testing.B) {
    for i := 0; i < b.N; i++ {
        findConcurrent(runtime.NumCPU(), "test", docs)
    }
}
```

列表13是基准测试方法。下面是只有一个操作系统/硬件线程对所有Goroutines可用时的结果。串行的版本只使用了一个Goroutines，而并发的版本使用了runtime.NumCPU个，在我的机器上也就是8个Goroutines.这种情况下，并发的版本并没有利用并行来提高并发。

#### Listing 14

```
10 Thousand Documents using 8 goroutines with 1 core
2.9 GHz Intel 4 Core i7
Concurrency WITHOUT Parallelism
-----------------------------------------------------------------------------
$ GOGC=off go test -cpu 1 -run none -bench . -benchtime 3s
goos: darwin
goarch: amd64
pkg: github.com/ardanlabs/gotraining/topics/go/testing/benchmarks/io-bound
BenchmarkSequential      	       3	1483458120 ns/op
BenchmarkConcurrent      	      20	 188941855 ns/op : ~87% Faster
BenchmarkSequentialAgain 	       2	1502682536 ns/op
BenchmarkConcurrentAgain 	      20	 184037843 ns/op : ~88% Faster
```

列表14中的基准测试结果表明：在只有一个操作系统/硬件线程的情况下，并发版本比串行版本快大约87%～88%。这正是我所期望的，因为所有Goroutines高效地共享同一个操作系统/硬件线程。当每个Goroutines调用read方法时，很自然的发生上下文切换，以使单个操作系统/硬件线程能够完成更多的工作。

下面是使用并行进行并发的基准测试结果。

#### Listing 15

```
10 Thousand Documents using 8 goroutines with 1 core
2.9 GHz Intel 4 Core i7
Concurrency WITH Parallelism
-----------------------------------------------------------------------------
$ GOGC=off go test -run none -bench . -benchtime 3s
goos: darwin
goarch: amd64
pkg: github.com/ardanlabs/gotraining/topics/go/testing/benchmarks/io-bound
BenchmarkSequential-8        	       3	1490947198 ns/op
BenchmarkConcurrent-8        	      20	 187382200 ns/op : ~88% Faster
BenchmarkSequentialAgain-8   	       3	1416126029 ns/op
BenchmarkConcurrentAgain-8   	      20	 185965460 ns/op : ~87% Faster
```

列表15中的基准测试结果表明额外的操作系统/硬件线程并没有带来更好的性能表现。


## 总结

这篇文章的目的当你考虑一个工作负载是否需要并发时，提供语义上的指导。我尝试提供不同算法类型和工作负载的例子，来向你展示语义上的不同和你需要做的不同的工程决策。

你能清楚的看到，IO密集型的工作负载，并行并不会带来明显的性能提升。CPU密集型则相反，例如冒泡排序，使用并发不会提高性能，只会增加复杂度。确定你的工作负载是否适合并发，然后确定你的工作负载使用正确的语义是很重要的。