---
title: "Go语言调度（一）：系统调度器"
date: 2021-02-01T16:01:24+08:00
draft: false
---

# Go语言调度（一）：系统调度器

## 序言

这是一个由三部分组成的系列文章，翻译而来，[原文出处请戳](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)。 将提供对Go语言调度器背后的机制和语义的理解。这是第一篇，这篇文章将着重于操作系统调度器。

索引：

1）Go语言调度（一）：系统调度器

2）Go语言调度（二）：Go调度器

3）Go语言调度（三）：并发

## 简介
归功于对操作系统sheduler的机制的一致性，Go sheduler的设计及行为使多线程Go程序的效率、性能更好。如果多线程Go程序的机制和调度器的机制工作方式不一样，那么一切都毫无意义。了解操作系统及Go的sheduler是如何工作的，对正确的设计多线程程序十分重要。

这系列文章将专注于讨论shedulers高级的机制及语义。我将提供足够的信息来可视化sheduler是如何工作的，以使你能够作出更好的工程决策。尽管一个好的多线程应用程序是由多方面条件决定的，但sheduler的机制和语义是所需基础知识的关键部分。

## 操作系统scheduler
操作系统的shedulers由一系列复杂的软件组成。它们必须考虑运行时的硬件的布局和设置，包括但不限于多个处理器及内核，CPU缓存及NUMA。没有这些，shedulers无法尽可能高效。幸运的是，你仍然可以建立一个良好的操作系统schedulers如何工作的思维模型，而无需深入研究这些主题。

你的程序实际上是一连串需要顺序执行的机器指令。为此，操作系统使用了线程的概念。线程的工作就是负责顺序执行分配给它的指令。执行动作将持续到没有指令可以继续执行，这就是为什么我把线程叫做“执行路径”（a path of execution）。

每个运行的程序都会创建一个进程，并且为每个进程分配一个初始线程。线程有创建更多线程的能力。所有这些不同的线程彼此独立运行，并且调度决策是在线程级别而不是在进程级别作出的。线程可以并发运行（每个线程轮流在同一个核上运行），也可以并行运行（每个线程在不同的内核上同时运行）。线程还维护自己的状态，以使线程可以安全的、本地的、独立的执行其指令。

操作系统sheduler负责保证当有线程可以执行的时候，内核不会处于空闲状态。它还必须创建一种表象，即所有可以执行的线程都在同时执行。在创建这种表象的过程中，sheduler还必须让具有高优先级的线程优先运行，但是低优先级的线程也不能完全没有运行时间。sheduler还必须通过作出快速且明智的决定，以尽可能的减小调度延迟。

要实现这一点，需要大量的算法。幸运的是，我们有数十年的工作和行业经验。为了更好的理解这些，描述和定义一些重要的概念是很不错的。

## 执行指令

程序计数器（program counter， PC）有时被称作指令指针（instruction pointer， IP），它允许线程跟踪下一个要执行的指令。在大多数处理器中，PC指向下一条需要执行的指令，而不是当前指令。

#### Figure 1
![IP](https://raw.githubusercontent.com/realSinged/img/master/instruction_pointer.jpeg)

如果你曾经看过Go程序的堆栈跟踪，你可能会注意到每行末尾的这些小的16进制数字，在Listing1中查找+0x39和+0x72。

#### Listing 1

    goroutine 1 [running]:
       main.example(0xc000042748, 0x2, 0x4, 0x106abae, 0x5, 0xa)
           stack_trace/example1/example1.go:13 +0x39                 <- LOOK HERE
       main.main()
           stack_trace/example1/example1.go:8 +0x72                  <- LOOK HERE

这些数字代表PC值从各自函数顶部的偏移量。+0x39偏移量表示如果程序没有panic，则线程将执行的下一条指令。+0x72偏移量是主函数中的下一条指令，如果控制指令正好返回到那个函数。更重要的是，该指针之前的指令告诉你正在执行的指令是什么。

请看Listing2中的程序造成了Listing1的异常。
#### Listing2
    https://github.com/ardanlabs/gotraining/blob/master/topics/go/profiling/stack_trace/example1/example1.go

    07 func main() {
    08     example(make([]string, 2, 4), "hello", 10)
    09 }


    12 func example(slice []string, str string, i int) {
    13    panic("Want stack trace")
    14 }

十六进制数+0x39表示示例函数内指令的PC偏移量，该偏移量比该函数的起始指令低57个字节(基数10)。在下面的Listing3中，您可以看到二进制文件中示例函数的objdump。找到在底部的第12条指令。注意，该指令上面的代码行是对panic的调用。

#### Listing3
    $ go tool objdump -S -s "main.example" ./example1
    TEXT main.example(SB) stack_trace/example1/example1.go
    func example(slice []string, str string, i int) {
    0x104dfa0		65488b0c2530000000	MOVQ GS:0x30, CX
    0x104dfa9		483b6110		CMPQ 0x10(CX), SP
    0x104dfad		762c			JBE 0x104dfdb
    0x104dfaf		4883ec18		SUBQ $0x18, SP
    0x104dfb3		48896c2410		MOVQ BP, 0x10(SP)
    0x104dfb8		488d6c2410		LEAQ 0x10(SP), BP
	    panic("Want stack trace")
    0x104dfbd		488d059ca20000	LEAQ runtime.types+41504(SB), AX
    0x104dfc4		48890424		MOVQ AX, 0(SP)
    0x104dfc8		488d05a1870200	LEAQ main.statictmp_0(SB), AX
    0x104dfcf		4889442408		MOVQ AX, 0x8(SP)
    0x104dfd4		e8c735fdff		CALL runtime.gopanic(SB)
    0x104dfd9		0f0b			UD2              <--- LOOK HERE PC(+0x39)

备注:PC是下一个指令，而不是当前指令。Listing3是一个很好的基于amd64指令的示例，这个Go程序的线程负责按顺序执行这些指令。

## 线程状态

另一个重要的概念是线程状态，它规定了sheduler在线程中扮演的角色。线程可以处于三种状态之一:等待（Waitiing）、可运行（Runnable）或正在执行（Running）。

**等待**:表示线程停止并等待以便继续。原因可能是等待硬件(磁盘、网络)、操作系统(系统调用)或同步调用(原子调用、互斥锁)。这些类型的延迟是糟糕性能的根本原因。

**可运行**:表示线程需要在内核上运行时间，这样它就可以执行分配给它的机器指令。如果有很多线程需要时间，那么线程必须等待更长时间才能获得时间。此外，任何给定线程获得的单独时间量也会缩短，因为更多的线程会争用时间。这种类型的调度延迟也可能是性能不佳的一个原因。

**正在执行**:这表示着线程已经被放置在一个核心上，并且正在执行它的机器指令。与应用程序相关的工作正在完成。这是每个人都想要的。

## 线程工作类型

线程可以做两种类型的工作。第一个称为cpu绑定（CPU-Bound），第二个称为io绑定（IO-Bound）。

**cpu绑定**:这种工作不会导致线程处于等待状态。这是一项不断计算的工作。计算Pi的第n位数字的线程是cpu绑定的。

**io绑定**:这是将导致线程进入等待状态的工作。这种工作包括通过网络请求访问资源或对操作系统进行系统调用。需要访问数据库的线程将是io绑定的。我将包括导致线程等待的同步事件(互斥锁、原子事件)，也作为这个类别的一种。

## 上下文切换

如果您运行在Linux、Mac或Windows上，那么您运行在一个具有抢占式sheduler的操作系统上。这意味着一些重要的事情。

首先，这意味着sheduler在任何给定时间选择运行哪些线程时是不可预测的。线程优先级和事件一起(比如在网络上接收数据)使得无法确定sheduler将选择什么时候做什么。

其次，这意味着你永远不能基于你有幸经历的、但不能保证每次都会发生的行为来编写代码。这很容易让你自己去想这是有保证的行为（但实际并不是），因为我已经见过类似的情况1000次了。如果需要确定性，则必须在应用程序中控制线程的同步和编排。

在内核上交换线程的物理行为称为上下文切换。当sheduler将一个执行线程从核心中取出，并用一个可运行线程替换它时，就会发生上下文切换。从运行队列中选择的线程进入执行状态。被替代的线程可以移动回可运行状态(如果它仍然有运行的能力)，或者移动到等待状态(如果由于请求的io绑定类型而被替换)。

上下文切换是昂贵的，因为在内核上交换线程需要花费很多时间。上下文切换期间的当前延迟量取决于不同的因素，但它在~1000到~1500纳秒之间是合理的。考虑到硬件应该能够(平均)执行12条指令每个核每纳秒，上下文切换可能会花费你~12k到~18k的指令延迟。从本质上说，您的程序在上下文切换期间失去了执行大量指令的能力。

如果你有一个io密集型的程序，那么上下文切换将是一个优势。一旦一个线程进入等待状态，另一个处于可运行状态的线程就会取代它的位置。这使得内核始终在工作。这是调度最重要的方面之一，如果有工作(处于可运行状态的线程)要做，不要让内核处于空闲状态。

如果您的程序是CPU密集型的，那么上下文切换将成为性能噩梦。因为线程总是有工作要做，所以上下文切换会阻止工作的进行。这种情况与io密集型工作形成了鲜明的对比


## 少即是多

在早期，处理器只有一个核心，调度并不太复杂。因为只有一个处理器和一个核心，所以在任何给定时间只能执行一个线程。定义一个调度器周期，并尝试在这段时间内执行所有可运行线程，只用将调度周期除以需要执行的线程数。

例如，如果您将调度程序的周期定义为1000毫秒(1秒)，并且您有10个线程，那么每个线程都有100毫秒。如果你有100个线程，每个线程得到10毫秒。但是，如果有1000个线程会发生什么呢?给每个线程一个1ms的时间片是行不通的，因为花费在上下文切换上的时间百分比与花费在应用程序工作上的时间相比将会非常明显。

你所需要的是对给定的时间片的长度设置一个限制。在上面的场景中，如果最小时间片是10ms，而你有1000个线程，那么调度器周期需要增加到10000ms(10秒)。如果有10,000个线程，那么现在的调度器周期是100000ms(100秒)。在10,000个线程的情况下，最小时间片为10ms，如果每个线程都使用它的完整时间片，那么在这个简单的示例中，所有线程运行一次需要100秒。

请注意，这只是一个非常简单的场景。在制定调度决策时，调度程序还需要考虑和处理更多的事情。您可以控制应用程序中使用的线程数。当有更多的线程需要考虑，io绑定的工作发生时，会有更多的混乱和不确定性行为。任务需要更长的时间来调度和执行。

这就是为什么规则是“少即是多”。处于可运行状态的线程更少，意味着更少的调度开销和每个线程执行时间越多。处于可运行状态的线程越多，每个线程占用的时间就越少。这也意味着随着时间的推移，你完成的工作也会变少。

## 找到平衡

为了使应用程序获得最佳吞吐量，您需要在内核数量和线程数量之间找到一个平衡。当谈到管理这种平衡时，线程池是一个很好的答案。我将在第二部分展示，这在Go中不再是必要的。我认为这是Go为简化多线程应用程序开发所做的一件好事。

在Go编码之前，我在NT上用c++和c#编写代码。在那个操作系统上，使用IOCP (IO完成端口)线程池是编写多线程软件的关键。作为一名工程师，您需要计算出需要多少线程池，以及任何给定池的最大线程数，以便在给定的核数下最大化吞吐量。

当编写和数据库交互的web程序时，在NT系统上，每个核3个线程总是能提供最佳吞吐量。换句话说，每个核3个线程能最小化上下文切换的开销同时最大化执行时间。当创建IOCP线程池的时候，我知道给每个核分配1～3个线程。

如果我每核使用两个线程，完成所有工作的时间将会变长，因为我有空闲时间，而本可以用来完成工作。如果我每核使用4个线程，完成工作也需要更长的时间，因为上下文切换会有更多的延迟。每核3个线程，无论出于什么原因，似乎总是NT上的神奇数字。

如果你的服务做很多不同类型的工作怎么办？这可能造成不同且不一致的延迟。也许它还创建了许多不同的系统级时间需要处理。不太可能找到一个适用于所有不同工作的神奇数字。在使用线程池来调优服务性能时，找到正确一致的配置会变得非常复杂。


## Cache Lines

从主存访问数据有很高的延迟成本(~100到~300个时钟周期)，因此处理器和核心都有本地缓存，将数据保存在需要数据的硬件附近，以便线程访问。从缓存中访问数据的成本更低(~3到~40个时钟周期),成本差异取决于缓存类型。。今天，性能的一个方面是如何减少数据访问延迟，将其高效地放入处理器。编写改变状态的多线程应用程序需要考虑缓存系统的机制。

#### Figure2
![cpu_cache](https://raw.githubusercontent.com/realSinged/img/master/cache_line_false_share.png)

处理器和主存之间使用cache lines来交换数据。cache line是一个在主存和缓存系统之间交换的64字节的内存块。每个核心都有它自己需要的cache line的副本，这意味着硬件使用值语义。这就是为什么在多线程应用中内存的变化会造成性能噩梦。

当多个并行运行的线程访问相同的数据，或者接近彼此的的数据时，他们将访问同一cache line上的数据。运行在任何内核上的任何线程都将获得同一cache line的副本。

Figure 3
![false_sharing](https://raw.githubusercontent.com/realSinged/img/master/cache_line_false_share.png)

如果给定核心上的一个线程对其cache line的副本进行了更改，那么通过硬件的魔法，同一cache line的所有其他副本都必须被标记为dirty。当一个线程尝试对dirty cache line进行读或写访问时，需要对主存进行访问(~100到~300个时钟周期)才能获得cache line的新副本。

也许在2核处理器上这不是什么大问题。但是在32核处理器上运行32个线程，在同一cache line上访问和改变数据呢？如果系统有两个16核的物理处理器呢？因为增加的处理器之间的通信延迟，这将变得更糟糕。应用程序将在内存中翻腾，性能将会非常糟糕，而你很可能根本不知道为什么。

这就是所谓的缓存一致性问题，也会带来错误共享等问题。在编写会改变共享状态的多线程应用程序时，必须考虑缓存系统。

## 调度决策场景

假设我请你根据我提供的高级信息来编写系统调度器。考虑一下你必须考虑的一个场景，这是调度程序在做出调度决策时必须考虑的许多有趣的事项之一。

启动应用程序，创建主线程并在核心1上执行。当线程开始执行指令时，因为需要数据，cache line将被检索。线程现在为了某些并发处理决定创建一个新线程。问题来了。

一旦新的线程创建完成，并且准备好运行。OS sheduler应该:
1. 将主线程上下文从核心1上切换下来？这么做可以提高性能，因为新线程需要的正好是已经被缓存的数据的几率非常高。但是主线程没有得到它全部的时间片。
2. 新线程是否在主线程的时间片完成之前等待核心1变得可用?线程没有运行，但一旦线程启动，获取数据的延迟将被消除。
3. 新线程是否等待下一个可用的核心？这将意味着所选核心的cache lines将被刷新，检索和重复，从而导致延迟。但线程将会更快地启动，主线程也会完成它的时间片。

有趣吧？这些都是操作系统sheduler在做调度决策时都需要考虑的问题。幸运的时，我不是编写操作系统sheduler的人。我能告诉你的是，如果有一个空闲核心，它就可以被使用。你可以使线程在可以运行的时候运行。

## 总结

这个系列文章的第一篇介绍了在编写多线程应用程序时必须要考虑的线程和操作系统scheduler，这些也是Go sheduler需要考虑的事情。在下一篇文章中，我将描述Go sheduler的语义，以及如何同这些事情相关联。最后，通过运行几个程序，您将看到所有这些操作。