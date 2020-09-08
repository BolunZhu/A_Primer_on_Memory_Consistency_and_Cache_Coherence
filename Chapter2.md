# Chapter 2 Coherence Basic 
# 缓存一致性基础

## 2.1 baseline system model
我们在本书考虑多个处理器核共享内存的系统，所有的核都可以访问所有的(物理)地址。我们的baseline系统模型包括一个多核芯片和off-chip main memory，具体如图2.1。

![2.1](/assets/2.1.png)

多核处理器由多个单线程核组成，每个核有自己的私有数据缓存data cache，以及一个被所有核共享的Last level cache(LLC)。在本书中，每当我们使用缓存cache时，我们指的是每个核所独有的data cache而非LLC。每个核的cache通过物理地址访问，并且使用write-back写回策略。核与LLC之间通过一个interconnection network进行交流。LLC，虽然在片上on-chip但是逻辑上是一个“memory side cache”内存侧缓存，因此不产生任何级别的缓存一致性问题coherence issues。LLC逻辑上只是用来放在内存前面减少延迟并且增加内存的有效带宽。LLC还充当片上的内存控制器on-chip memory controller。

这个baseline系统忽略了很多常见但是对本书主题而言没有用的特性，包括指令cache，多层cache，多核共享缓存，虚拟地址cache，tlb，DMA等。这个系统同样忽略了多个多核处理器组成的系统，我们之后会讨论这些，但是目前这些内容将增加不必要的复杂性。

------
## 2.2 The problem: how incoherence could possibly occur
缓存不一致性只可能产生在一个基本的情况:多个参与者访问cache和memory。在现代系统中，这些参与者既可能是cores，也可能是DMA engines以及一些外部设备，他们可以读或/和写内存。    
>In modern systems, these actors are processor cores, DMA engines, and external devices that can read and/or write to caches and memory.

在本书中我们主要考虑处理器核作为参与者的情况，但是需要记住，其他的参与者也是存在的。

![2.2](/assets/2.2.png)

图2.2展示了一个简单的缓存不一致性的例子。一开始内存地址为A的数据值为42。随后core1 和core2各自将A load到自己的cache里面。在时刻3的时候，core1将A的值加1，导致A在core1的cache中的值加1，使得core2上A在cache上的值处于stale/incoherent缓存不一致的状态。

为了避免这种不一致，系统必须实现一个缓存一致性协议 _cache coherence protocol_，从而规避如在core1更新A的值的同时core2观测到旧值的情况。缓存一致性协议的具体设计和实现是第7到第9章的主要内容。

------
## 2.3 Defining coherence
上面的例子展示了一个直观的缓存不一致性的例子，多个参与者在同时观察到同一个数据不同值。这一章我们将精确的定义coherence缓存一致性。在许多论文和教科书中出现了许多对于coherence的定义，在这里我们使用我们更青睐的定义，这种定义洞察了缓存一致性协议的设计。
>we present the definition we prefer for its insight into the design of coherence protocols。

此外我们还将讨论一些其他的定义以及他们与我们的定义的关系。

我们关于coherence的定义的基础是 _single-writer-multiple-reader(SWMR)_ invariant。
对于任意给定的内存地址，在任何时刻[^1]，要么有一个核在写(也可能读)它，要么多个核在读它。因此不可能存在一个核在写一个内存地址的同时其他核在读/写同一个内存地址。另外一个看待这个定义的视角是，考虑每个内存地址的lifetime生命周期被分成很多步epochs，每一步中要么有一个核具有读-写权限，要么有多个核具有只读权限。

![2.3](/assets/2.3.png)

图2.3展示了一个给定的内存地址具有4个阶段，每个阶段都保持着SWMR不变性。

除了SWMR不变性以外，coherence还需要给定内存地址的值被正确传播。例如，即便遵循了SWMR，core2和core5也可能读到不一样的值。因此，一致性的定义必须用数-值不变量来增加SWMR不变量，该数-值不变量与值如何从一个阶段传播到下一个阶段有关。这个不变量指出，在一个阶段开始时，指定内存位置的值与它最后一个读写read-write阶段结束时的内存位置值相同。

对于这些不变量，还有其他的解释是等价的。一个著名的例子[<sup>5</sup>]用token解释了SMWR不变量。token不变量如下。对于每个内存位置，都存在一个固定数量的token令牌，其大小至少与核心的数量相同。如果一个核心拥有所有令牌，它可能会写入内存位置。如果一个核心有一个或多个令牌，它可以读取内存位置。因此，在任何给定的时间，不可能有一个内核正在写入内存位置，而另一个内核正在读或写它。
> __Coherence invariants__
>1. __Single-Writer, Multiple-Read (SWMR) Invariant__. For any memory location A, at any
given (logical) time, there exists only a single core that may write to A (and can also read it)
or some number of cores that may only read A.
>2. __Data-Value Invariant__. The value of the memory location at the start of an epoch is the same as the value of the memory location at the end of its last read–write epoch.

----------
### 2.3.1 Maintaining the coherence invariants
前面介绍的coherence invariants提供了关于缓存一致性协议如何工作的直观印象。绝大多数被称为“失效协议”的一致性协议都是显式地设计来维护这些不变量的。如果一个核想要读取一个内存位置，它会向其他核心发送消息，以获取该内存位置的当前值，并确保没有其他核心以读写状态缓存内存位置的副本。这些消息结束任何活动的读写epoch并开始只读epoch。如果一个核心想要写入内存位置，它会向其他核心发送消息，以获取该内存位置的当前值（如果它还没有有效的只读缓存副本），并确保没有其他内核在只读或读写状态下缓存了该内存位置的副本。这些消息结束任何活动的读写或只读epoch，并开始新的读写epoch。这本初级读物中关于缓存一致性的章节（第6-9章）对无效协议的抽象描述进行了大量扩展，但基本的直觉印象保持不变。

--------
### 2.3.2 The Granularity of Coherence
核可以执行不同粒度的load和store，通常从1到64字节不等。理论上，coherence可以在最细的load/store粒度上执行。然而，在实践中，coherence通常保持在缓存块的粒度上。也就是说，硬件在每个缓存块的基础上强制执行一致性。实际上，SWMR不变量很可能是，对于任何内存块，要么只有一个writer，要么有一些reader。在典型的系统中，一个核不可能写入一个块的第一个字节，而另一个核则写入该块中的另一个字节。尽管缓存块粒度是常见的，而且这也是我们在本初级读物的其余部分中所假设的，但是我们应该知道，有一些协议在更细和更粗的粒度上保持了coherence.
> __Sidebar: Consistency-Like Definitions of Coherence__
> 
>  我们关于coherence的定义使用两个不变量：一个关于不同核对同一个内存地址的访问权限，另外一个关于数据的值在不同的核之间如何传递。然而 __还存在一类像consistency一样的coherence的定义__，关注load和store，类似内存一致性模型定义架构可见的load和store顺序一样。
> 
> 其中一个类似内存一致性的coherence定义与Sequential Consistency有关。我们将在第3章深入介绍SC，SC指定系统必须以符合每个线程的程序顺序的全序执行所有线程的加载和存储到所有内存位置。每次加载都按该全序获取最近存储的值。一个类似于SC定义的coherence定义是，一个coherence的系统必须以尊重每个线程的程序顺序的全序执行所有线程的加载和存储到单个内存位置。这个定义突出coherence和consistency之间的一个重要区别：coherence是基于每个内存位置指定的，而consistency是针对所有内存位置指定的。
> 
> coherence的另一个定义[<sup>1</sup>] [<sup>2</sup>]用两个不变量定义了一致性：
> - (1) 每个store最终对所有核可见
> - (2) 对同一内存位置的写入被序列化（即，所有核观察到相同的顺序）。
> 
> IBM在Power架构[<sup>5</sup>]中采用了类似的观点，部分原因是为了便于实现这样的实现：一个核的一系列store可能已经到达了一些核（这些核的load可以看到它们的值），而不是其他核。
> 
> 还有一个coherence的定义来自于 Hennessy and Patterson [<sup>3</sup>].由3个不变量组成：
> - (1) 一个核对地址A的读到的值为这个核上一次写到A的值，除非两次读和写之间有别的核写了A
> - (2) 读A(L)得到另一个核写(S)的值，只有在这L和S _are sufficiently separated in time_ 并且L和S之间没有别的store发生
> - (3) 对同一内存位置的写入被序列化（即，所有核观察到相同的顺序）。
> 
> 这个定义也很直观，但是问题在于对 _are sufficiently separated in time_ 缺乏精确的定义。
> 
> __这些类似consistency的coherence定义，和2.3给出的coherence定义都是valid合理的__，都可以很容易地用作验证给定协议是否具有缓存一致性的规范。一个正确的缓存一致性协议可以满足任意一种valid合理的定义。__然而这类像consistency的coherence定义缺乏对缓存一致性协议架构的直观表现__。对架构师而言,设计缓存一致性协议是用于规范核对内存地址的访问，__因此2.3提到的定义对于架构师而言更加直观__。

----
### 2.3.3 The Scope of Coherence
无论我们选择哪个定义，一致性的定义都有一个特定的范围，架构师必须知道它何时适用，何时不适用。我们现在讨论两个重要的范围问题：
> * coherence适用于保存来自共享地址空间的块的所有存储结构。这些结构包括一级数据缓存、二级缓存、共享末级缓存（LLC）和主内存。这些结构还包括一级指令缓存和转换查询缓冲区（tlb）[^2]
> * coherence与体系结构无关（即，coherence在架构上不可见）。严格地说，如果系统遵循指定的内存一致性模型，它可能是incoherent的，并且仍然是正确的。虽然这个问题看起来像是一种智力上的好奇（即，很难想象一个实际的系统是consistent但incoherent的），但它有一个非常重要的推论：内存一致性模型没有对缓存一致性或用于执行缓存一致性的协议施加明确的限制。尽管如此，正如在第3章到第5章中所讨论的，许多一致性模型实现都依赖于某些共同的缓存一致性属性来保证正确性，这就是为什么我们在继续讨论内存一致性模型之前在本章中介绍了缓存一致性。


---------
## 2.4 参考文献

<div id="refer-anchor-1"></div>

[1] [K. Gharachorloo. Memory Consistency Models for Shared-Memory Multiprocessors. PhD thesis, Computer System Laboratory, Stanford University, Dec. 1995]

<div id="refer-anchor-2"></div>

[2] [K. Gharachorloo, D. Lenoski, J. Laudon, P. Gibbons, A. Gupta, and J. Hennessy. Memory
Consistency and Event Ordering in Scalable Shared-Memory. In Proceedings of the 17th
Annual International Symposium on Computer Architecture, pp. 15–26, May 1990.]

<div id="refer-anchor-3"></div>

[3] [J. L. Hennessy and D. A. Patterson. Computer Architecture: A Quantitative Approach.
Morgan Kaufmann, fourth edition, 2007.]

<div id="refer-anchor-4"></div>

[4] [IBM. Power ISA Version 2.06 Revision B. http://www.power.org/resources/downloads/PowerISA_V2.06B_V2_PUBLIC.pdf, July 2010.](http://www.power.org/resources/downloads/PowerISA_V2.06B_V2_PUBLIC.pdf)

<div id="refer-anchor-5"></div>

[5] [M. M. K. Martin, M. D. Hill, and D. A. Wood. Token Coherence: Decoupling Performance and Correctness. In Proceedings of the 30th Annual International Symposium on Computer Architecture, June 2003. doi:10.1109/ISCA.2003.1206999](doi:10.1109/ISCA.2003.1206999)
  
-------
# 译者总结
本章主要给出了coherence的具体定义，为了定义coherence，本章首先介绍了一个baseline系统。使用一个interconnection network连接每个核的独享cache。随后给出了一个比较好的coherence定义，使用两个不变量，SWMR和data-value。随后讨论了一些其他的coherence模型，这些模型也都是正确的，但是要么不够精确，要么不能提供架构上的直观。




[^1]: SWMR invariant只在 _logical time_ 维持，而非物理时间。这个微妙的区别导致许多优化看起来违背了(实际上没有)这一不变性。我们将在后续的章节中讨论。
[^2]: 有些时候TLB里面的映射也不是严格意义上的共享内存中的拷贝