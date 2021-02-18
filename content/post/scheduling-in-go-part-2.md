---
title: "Go语言调度（二）：Go调度器"
date: 2021-02-08T10:43:09+08:00
draft: false
---

## 序言
这是一个由三部分组成的系列文章，翻译而来，[原文出处请戳](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)。 将提供对Go语言调度器背后的机制和语义的理解。这是第二篇，这篇文章将着重于Go调度器。

索引：

1）Go语言调度（一）：系统调度器

2）Go语言调度（二）：Go调度器

3）Go语言调度（三）：并发

## 简介
在本系列的第一篇文章中，我阐述了关于操作系统调度器的一些方面，我认为这些对于理解和欣赏Go调度器的语义是非常重要的。在这篇文章中，我将在语义层面解释Go调度器是如何工作的，并将重点放在高级别的行为上。Go调度器是一个复杂的系统，在这篇文章中，我们将忽略一些小的细节，并将重点放在建立一个调度器是如何工作和表现的模型上，这些将会让你做出更好的工程决策。

## 你的程序开始运行了

当你的Go程序启动后，它将会为主机上识别的每个虚拟核心提供一个逻辑处理器（P）。如果你有一个包含多个硬件线程(超线程)的处理器，每个硬件线程都将作为一个虚拟核心呈现给你的Go程序。为了更好的理解这一点，请看我的MacBook Pro的系统报告。

#### Figure 1
![94_figure1.png](https://upload-images.jianshu.io/upload_images/13462240-b546277d3b846c9f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如你所见，我有一个包含4个物理核心的处理器。这个报告没有展示每个物理核心的硬件线程数。Intel Core i7处理器拥有超线程，这意味着每个物理核心有2个硬件线程。所以这将报告给Go程序一共有8个虚拟核心可以用来并行执行操作系统线程。

为了测试这一点，请看下面的程序：

#### List 1

    package main

    import (
	    "fmt"
	    "runtime"
    )

    func main() {

        // NumCPU returns the number of logical
        // CPUs usable by the current process.
        fmt.Println(runtime.NumCPU())
    }

当我在本机上运行这个程序时，NumCPU()函数调用的结果将是值8。我在我的机器上运行的任何Go程序都将得到8个P。

每个P都被分配了一个OS线程（“M”）。M代表机器（mechine）。这个线程由操作系统管理并且操作系统负责将这个线程放在一个核心上执行， 正如第一篇文章解释的那样。 这意味着当我在本机上运行Go程序时，我有8个OS线程可以执行我的工作，每一个线程都绑定一个P。

每个Go程序都有一个初始Goroutine（“G”），这就是Go程序的执行路径。Goroutine本质上就是Coroutine，但因为是Go语言，所以我们将字母“C”替换成了“G”，因此得到了词语Goroutine。你可以认为Goroutine就是应用程序级的线程，并且它在多个方面和线程类似。Goroutine在M上进行上下文切换，正如线程在核心上进行上下文切换一样。

最后一个难题是运行队列。 Go调度器中有两种不同的运行队列： 全局运行队列（the Global Run Queue， GRQ）和本地运行队列（the Local Run Queue， LRQ）。每个P都被指定了一个LRQ，这个LRQ管理着在P的上下文中执行的Goroutine队列，这些Goroutines轮流在绑定了这个P的M上进行上下文切换。GRQ是还没有分配P的Goroutine队列。有一个将Goroutine从GRQ移动到LRQ的过程，我们将在后面讨论。

#### Figure 2

![94_figure2.png](https://upload-images.jianshu.io/upload_images/13462240-67a88c18e9db6bfe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 协作式调度器

正如我们在第一篇文章中讨论的那样，操作系统调度器是一种抢占式调度器。本质上，这意味着您无法预测调度器在任何给定时间将要做什么。核心将会做决定，一切都是不确定的。运行在操作系统之上的应用程序无法通过调度控制内核内发生的事情，除非它们利用像原子指令（atomic instructions）和互斥调用（mutex calls）这样的同步原语。

Go调度器是Go运行时（runtime）的一部分，而Go运行时是构建在用户程序中的。这意味着Go调度器是运行在用户空间里（user space），而不是在内核空间里。当前的Go调度器的实现是协作式调度，而不是抢占式调度。协作式调度意味着调度器需要在代码中安全的地方产生定义良好的用户空间事件来进行调度决策。

Go协同式调度器的绝妙之处在于它看起来像是抢占式的。你无法预测Go调度器将会做什么。这是因为调度器做的决策不是由开发人员决定的，而是由Go运行时决定的。把Go调度器看作是抢占式的非常重要，因为调度器是不确定的。

## Goroutine状态

就像线程一样，Goroutine也有3个相同的高级状态。这些状态规定了Go调度器在任何给定的Goroutine中所扮演的角色。Goroutine可以处于3种状态： **等待（Waiting）**、**可运行（Runnable）**或 **正在执行（Executing）**。

**Waiting**： 这意味着Goroutine停止并等待某些事情以便继续。原因可能是等待操作系统(系统调用)或同步调用(原子操作和互斥操作)。这些类型的延迟是糟糕性能的根本原因。

**Runnable**：这意味着Goroutine需要M上的时间，这样它就可以执行它所分配的指令。如果你有很多需要时间的Goroutines，那么Goroutines必须等待更长时间来获得时间。此外，因为更多的Goroutine竞争时间，任何给定的Goroutine获得的时间都缩短了。这种类型的调度延迟也可能是性能不佳的一个原因。

**Executing**：这意味着Goroutine已经被放置在一个M上，并且正在执行它的指令。与应用程序相关的工作正在完成。这是每个人都想要的。

## 上下文切换

Go调度器需要定义良好的用户空间事件，这些事件发生在代码中的安全点，以便进行上下文切换。这些事件和安全点在函数调用中出现。函数调用对Go调度器的运行状态至关重要。现在(使用Go 1.11或更低版本)，如果运行任何没有进行函数调用的[紧凑循环](https://stackoverflow.com/a/2213001/9228186)，就会导致调度程序和垃圾收集的延迟。函数调用的时长必须在合理的时间范围内进行，这一点非常重要。

注：在1.12版本中有一个提议被接受，它在Go调度器中应用非合作抢占技术，以允许紧凑循环的抢占。

在Go程序中，有四类事件的发生，可以允许调度器做出调度决策。但这并不意味着发生这些事件时总是会做出调度，只是说调度器得到了调度机会。
* 关键字go的使用
* 垃圾回收
* 系统调用
* 同步及编排（Synchronization and Orchestration）

### 关键字go的使用

关键字go是用来创建goroutines的，一旦创建了一个新的goroutine，它就给调度器一个机会来做出调度决策。

### 垃圾回收

由于GC使用自己的goroutines集运行，并且这个goroutines需要运行在M上。这将导致GC产生大量调度混乱。然而，调度器很聪明，知道goroutine正在做什么，它将利用这种能力来做出明智的决定。其中一个明智的决定是在GC期间将想要接触堆的和不想要接触堆的goroutines进行上下文切换。当GC运行时，许多调度决策将会被做出。

### 系统调用

如果一个goroutine进行系统调用，这将会导致这个goroutine阻塞M，有时候调度器能够将这个goroutine从M上切换下来，并切换一个新的goroutine上去。然而也有的时候需要一个新的M来执行在P中排队的goroutines。接下来的部分将更加详细的解释这些情况下，调度器是如何工作的。

### 同步及编排

如果一个原子的、互斥的或者channel操作调用将导致goroutine阻塞，那个调度器就可以切换一个新的goroutine来运行。一旦这个goroutine准备好了，它就可以重新排队，并最终切换到M上继续运行。


## 异步系统调用

当运行的操作系统具有异步处理系统调用的能力时，可以通过使用称之为网络轮询器（network poller）的东西来更有效的处理系统调用。这是通过使用kqueue（MacOS），epoll（Linux）或iocp（Windows）在不同的操作系统上完成的。

我们今天使用的许多操作系统都可以异步地处理基于网络的系统调用。这也是网络轮询器名称的来源，因为它的主要用途是处理网络操作。通过使用网络轮询器进行网络系统调用，调度器可以防止goroutines在进行这些系统调用时阻塞M。这能够让M执行P的LRQ中其他的goroutines，而不需要创建新的M。这有助于降低操作系统上的调度负载。

了解其工作原理的最好方法是运行一个示例。

#### Figure 3
![94_figure3.png](https://upload-images.jianshu.io/upload_images/13462240-d6987f215a940944.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

图3是基本的调度示图。G1正在M上被执行，LRQ中还有3个G等待获取他们在M上的时间。网络轮询器是空闲的，无事可做。

#### Figure 4
![94_figure4.png](https://upload-images.jianshu.io/upload_images/13462240-cf0b473dbe6c4799.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在图4中，G1希望进行网络调用，因此将G1移动到了网络轮询器，这里将会处理异步网络调用。M现在就可以执行来自LRQ上的另外一个G。在本例中，G2被切换到了M上。

#### Figure 5
![94_figure5.png](https://upload-images.jianshu.io/upload_images/13462240-721e925d17a3e3d6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在图5中，网络调用器完成了异步网络调用，G1被移回了P的LRQ队列中。一旦G1可以在M上进行上下文切换，它对应的Go相关代码就可以再次执行。这里最大的优势是，执行网络系统调用不需要额外的M。网络轮询器只有一个OS线程，它正在处理一个有效的时间循环。

## 同步系统调用

当Goroutine想要进行不能异步完成的系统调用时会发生什么？在这种情况下，网络轮询器不可用，进行系统调用的Goroutine将会阻塞M。这很不幸，但无法阻止这种情况发生。不能进行异步系统调用的一个例子就是基于文件的系统调用。如果你正在使用CGO，那么调用C函数也会阻塞M也是其中一种情况。

_注： Windows操作系统有进行基于文件的异步系统调用的能力。从技术上讲，在Windows上运行时，可以使用网络轮询器。_

让我们继续看看同步系统调用（如文件I/O）导致M阻塞时会发生什么。

#### Figure 6
![94_figure6.png](https://upload-images.jianshu.io/upload_images/13462240-57b1bdaa2ce517e9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

图6再次展示了基本的调度示图。但是这次G1将会进行阻塞M的同步系统调用。

#### Figure 7
![94_figure7.png](https://upload-images.jianshu.io/upload_images/13462240-6c422a16217d2ce5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在图7中，调度器能够识别G1导致M阻塞。此时，调度器将绑定了G1的M1从P中分离出来。然后调度器引入一个新的M2来为P服务。此时就可以从LRQ中选择G2，然后切换到M2上。如果一个M因为之前的转换已经存在，那么这个转换比创建一个新的M要快。

#### Figure 8
![94_figure8.png](https://upload-images.jianshu.io/upload_images/13462240-a9ef302c20c6f6f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在图8中，G1进行的阻塞系统调用完成了。此时G1能够移回LRQ中并且被P继续服务。M1被放在一旁，如果将来这种情况再次发生，M1可以继续使用。

## 工作窃取（Work Stealing）

调度器的另一面是它是一个窃取工作的调度器。这有助于在某些领域保持调度效率。首先你最不希望看到的是M进入等待状态，因为一旦发生，操作系统将把M从核心上切换下来。这意味着直到M被重新切换回核心之前，即使有G处于可运行状态，P也不能完成任何工作。工作窃取也有助于平衡多个P上G的数量，这样工作也能更好的分配和更高效的完成。

让我们来看一个例子。

#### Figure 9
![94_figure9.png](https://upload-images.jianshu.io/upload_images/13462240-cc1a57b4867fc7a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在图9中，多线程Go程序有两个P，各服务4个G， GRQ中有一个G。如果其中一个P很快的服务完了它的所有的G怎么办？

#### Figure 10
![94_figure10.png](https://upload-images.jianshu.io/upload_images/13462240-f13aad31e6a3b344.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在图10中， P1已经没有G可以执行，但是在P2的LRQ中和GRQ中依然有可执行的G。在这个时刻，P1就需要窃取工作。窃取工作的规则如下表所示。

#### Listing 2

```
runtime.schedule() {
    // only 1/61 of the time, check the global runnable queue for a G.
    // if not found, check the local queue.
    // if not found,
    //     try to steal from other Ps.
    //     if not, check the global runnable queue.
    //     if not found, poll network.
}
```

所以基于表2的规则，P1需要检查P2的LRQ，然后偷走一半的G。

#### Figure 11
![94_figure11.png](https://upload-images.jianshu.io/upload_images/13462240-caf3dee304183a66.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在图11中，P2半数的G被P1偷走了，现在P1可以执行这些G。

如果P2完成了所有G的服务，而P1的LRQ中什么都没有了，会发生什么？

#### Figure 12
![94_figure12.png](https://upload-images.jianshu.io/upload_images/13462240-b04675cef83e92b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在图12中，P2完成了它所有的工作，现在需要去窃取一些G。首先它会查看P1的LRQ，但是一个G也找不到。接下来，他会查看GRQ，在这里，它找到了G9。

#### Figure 13
![94_figure13.png](https://upload-images.jianshu.io/upload_images/13462240-855abf83780f7e25.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在图13中，P2从GRQ中窃取G9并开始执行工作。所有这些窃取工作巧妙的地方在于，它让所有M保持忙碌而不是空闲。这种工作窃取在内部被认为是旋转M。这种旋转还有其他好处，JBD在它的[博客](https://rakyll.org/scheduler/)中解释得很好。

## 实例

有了这些机制和语义，我想向你展示如何将所有这些结合在一起，从而允许Go调度器在同样的时间里执行更多的工作。想象一个用C编写的多线程程序，程序管理着两个线程，他们之间互相传递消息。

#### Figure 14
![94_figure14.png](https://upload-images.jianshu.io/upload_images/13462240-d776a1e1a0b21a7a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在图14中，有两个线程来回传递消息。线程1在核心1上进行上下文切换，现在正在执行，这允许线程1将消息发送给线程2。

_注意:消息如何传递并不重要。重要的是在编排过程中线程的状态。_

#### Figure 15
![94_figure15.png](https://upload-images.jianshu.io/upload_images/13462240-79b31130a472b0a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在图15中，一旦线程1完成消息发送后，就需要等待响应。这将导致线程1从核心1上切换下来，并移动到等待状态。一旦线程2收到消息通知，它就进入可运行状态。现在，操作系统可以执行上下文切换，并让线程2在核心上执行，而这个核心恰好是核心2。接下来，线程2处理消息并将一条新消息发送回线程1。

#### Figure 16
![94_figure16.png](https://upload-images.jianshu.io/upload_images/13462240-44402ef44b62be40.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在图16中，线程1接收到线程2的消息时，线程上下文再次切换。现在，线程2从执行状态切换到等待状态，而线程1上从等待状态切换到可运行状态，最后到执行状态，这允许它处理并返回新消息。

所有这些上下文切换和状态更改都需要时间来执行，这限制了工作完成的速度。由于每个上下文切换可能会导致大约1000纳秒的延迟，而硬件可能每纳秒执行12条指令，因此在这些上下文切换期间没有执行，约12K的指令不能执行。由于这些线程也可能在不同的核心之间跳跃，因此由于缓存未命中而导致额外延迟的几率也很高。

让我们看看以这个相同的例子，但使用goroutine和Go调度器代替是怎样的。

#### Figure 17
![94_figure17.png](https://upload-images.jianshu.io/upload_images/13462240-9e6a2cf6e1af975d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在图17中，有两个goroutine彼此来回传递消息。G1在M1上进行上下文切换，M1恰好运行在核心1上，这允许G1执行它的工作。G1的工作是将消息发送给G2。

#### Figure 18
![94_figure18.png](https://upload-images.jianshu.io/upload_images/13462240-abb533ffbe0714a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在图18中，一旦G1完成了消息的发送，它将需要等待响应。这将导致G1从M1上切换下来并移动到等待状态。一旦G2收到消息通知，它就会进入可运行状态。现在Go调度器可以执行上下文切换，并让G2在M1上执行，M1仍然在核心1上运行。接下来，G2处理该消息并将一条新消息发送回G1。

#### Figure 19
![94_figure19.png](https://upload-images.jianshu.io/upload_images/13462240-385988a95b0ecef8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在图19中，当G2发送的消息被G1接收时，再次进行上下文切换。现在，G2从执行状态切换到等待状态，G1从等待状态切换到可运行状态，最后回到执行状态，这允许它处理并返回新消息。

表面上看并没有什么不同。无论您使用线程还是Goroutines，都会发生相同的上下文切换和状态更改。然而，使用线程和Goroutines之间有一个主要的区别，乍一看可能不是很明显。

在使用goroutines的情况下，所有处理都使用相同的操作系统线程和核心。这意味着，从操作系统的角度来看，操作系统线程永远不会进入等待状态，一次也没有。因此，操作系统线程上下文切换中丢失的指令，在使用goroutine时不会丢失。

从本质上讲，Go把I/O阻塞工作变成了操作系统级别的cpu密集型工作。因为所有的上下文切换都发生在应用程序级别，所以我们不会像使用线程时那样，每次上下文切换都需要花费大约12K指令（平均）。在Go中，这些相同的上下文切换需要花费约200纳秒或约2.4K指令。Go调度器还有助于提高cache-line效率和NUMA。这就是为什么我们不需要比虚拟核心更多的线程。在Go中，相同的时间可以完成更多的工作，因为Go调度器试图使用更少的线程，在每个线程上做更多的工作，这有助于减少操作系统和硬件的负载。

## 总结

Go调度器在设计上考虑了操作系统和硬件工作的复杂性，这一点非常让人惊叹。在操作系统级别上，将IO/阻塞工作转变为CPU密集型工作的能力是我们利用更多的CPU的一大优势。这就是为什么你不需要比虚拟内核更多的操作系统线程。您可以合理地预期，每个虚拟核只需要一个OS线程就可以完成所有的工作(CPU和IO/阻塞限制)。这样做对于网络应用和其他不需要阻塞操作系统线程的系统调用也是可以的。

作为一名开发人员，你仍然需要了解你的应用程序正在处理的工作类型。你不能创造无限数量的Goroutines和期望惊人的性能。少总是多，但是通过理解这些Go-scheduler语义，您可以做出更好的工程决策。在下一篇文章中，我将探索以保守的方式利用并发性来获得更好的性能，同时仍然平衡您可能需要添加到代码中的复杂性。











