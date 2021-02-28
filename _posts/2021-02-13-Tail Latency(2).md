---
layout: page
title: Analyses and Optimizations for Tail Latency
tags: [Distribution]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## Tales of the Tail: Hardware, OS, and Application-level Sources of Tail Latency

### 0x00 基本内容

 这篇paper是一般的client-server模型下面服务的一些关于tail latency的分析。这里讨论的主要是一些请求排队带来的影响。排队的模型使用*A*/*S*/*c*的方式表示，A表示请求等待的分布，S表示服务时间的分布，而c表示处理请求的workers的数量。一般情况下要求处理请求的速度在较长时间内的平均大于请求达到的平均，否则会导致排队的队长无限增长。在paper的数据中，其测试了单核和多核的情况下，一个echo rpc server、memcached服务和nginx服务在不同情况下请求的tail latency。paper中给出的测试方式从请求的network packets从到达网卡被网卡驱动处理，进入到kernel的TCP/UDP协议栈处理逻辑，到调度应用线程到一个CPU核心上面运行，到应用线程使用syscall从kernel中读出取钱，到用户线程处理完成然后调用syscall回复数据，到数据包被网卡驱动通过网卡发送几个过程分别计时。使用的echo rpc server、memcached和nginx三个服务中，echo rpc server使用的是多线程的模式，即每个请求启动一个新的线程来处理，而memcached的模型是固定线程数量的处理方式，而nginx则是 event-driven的架构。得出了这样的一些基本结论：

* 在FIFO/先到先处理、LIFO/后到先处理，两种方式都放到一个全局的队列的排队方式下，FIFO的方式有更低的tail latency，LIFO方式的tail latency要差一些，但是其median latency，即中位数延迟表现更好。另外的两种模式是random worker和random request的一些统计，其中random worker的方式每个worker有一个FIFO队列，每个work处理自己队列里面的任务，而random request的方式为一个全局的队列，每个worker从这个全局的队列中取task(不考虑其任务添加进来的顺序)，两种random的方式其tail latency都比FIFO的更差，但是median latency更好。同样的结论在后面的一些ZygOS的论文上面也有提到。
* 在单核的情况下，其它的Background Processes是一个造成tail latency的主要原因之一。一般情况下user-level的进程有相同的优先级，在只有一个CPU核的情况下容易造成资源的争抢。考虑到一般的Linux Kernel使用CFS调度的方式，其调整优先级能够对tail latency影响很小。在使用Linux实时调度器的情况下，对tail latency的提升很明显。因为Linux总是会优先处理实时进程，而CFS这样的方式调度不同优先级进程/线程的时候，其执行顺序和优先级没必然关系。CFS使用的调度方式是一种非FIFO的方式，而调度实时进程的时候可以使用FIFO的方式。
* 在多核的情况下，rpc server表现出更好的tail latency，而memcached和nginx则没有。Paper中认为rpc server的模型更像是一个random request的方式，而其它两个更像是一个random worker的方式。使用一个global的queue可以有更好的tail latency表现。通过将使用的传输协议由UDP改为TCP，memcached也可以切换为类似于使用global queue方式的处理模型。而nginx要向改为类似这样的模型，paper中使用的方式是client端美发送20-40个请求之后关闭连接，然后重新连接。
* 另外影响tail latency的发明是interrupt。处理一个网络请求的时候一般的方式是network packets到达NIC的时候，网卡驱动发出一个interrupt，通知一个CPU核心来处理。默认的情况下，Linux会使用一个irqbalance的机制，在系统负载比较低的情况下，会讲中断分配给一种centralized CPU核心来处理，而在系统负载比较高的情况下，Linix会讲中断分发到各个CPU核心，以实现更好的load balance。这样在paper测试的情况下，中断可能来自任何一个CPU核心。这样的方式导致了两个问题：1. 中断导致了context-switch和cache pollution，影响性能；2. 处理不再是FIFO的方式。Paper通过制定一个CPU核心来处理中断的方式，而其它的几个核心处理请求，测试这个能够比较明显地改进tail latency。另外一个影响tail latency的因素是NUMA，最好能够避免跨NUMA Node的访问。
* 还有CPU的一些节省电力的行为也可能导致tail latency增加。比如CPU可能在idle的时候进入一种“C-state”的状态，以及降低CPU频率，更节省电力。C-state由不同的level，越高的level会使得CPU更多的components进入一种关闭的状态。Linux通过CPU utilization以及其它的一些因素来选择是CPU进入不同的C-state。从C-state“唤醒”CPU的行为，以及降低CPU频率也会造成更高的tail latency。一种优化方式避免CPU进入进入比较高的C-state。

## Managing Tail Latency in Datacenter-Scale File Systems Under Production Constraints

### 0x10 基本内容

 这篇Paper讨论的是HDFS上的一些tail latency的优化。HDFS基本架构沿用了GFS的设计，NameNode保持原数据，DataNode保持数据。HDFS写入的时候是单个writer的写入，一般只有append的操作。这里的环境设置的block大小为256MB，写入的时候使用64KB的packet以piepline的方式写入，为了提高性能会以64并发的方式写入。这样应该会形成一些窗口为64的滑动窗口式的操作。对于写入操作，通过对这里的数据进行Spearman相关系数的分析，其和disk queue length的相关性最大。这个和多个写入操作写一个disk造成的资源冲突有比较大的关系，而且每个server上的写入导致写入这个server的local buffer cache也会有影响。Read操作的tail latency也同样和disk queue length有比较大的关系。影响写入的另外一个因素是这里测试环境对DataNode设计的一个60MB/s的限流值。另外测试的CPU、网络和内存对tail latency影响相对较小。

* 在上面的的数据分析基础之上，优化tail latency使用 track latency, check for slowness, decide whether to act, 以及take action。即现通过track的方式测量到那些请求是延迟比较大的请求，然后是latencies消息来发现那些servers是slow的。作出那些servers是slow的判断之后，就是确定采用什么样的处理方式，比如fail fast for a write 或 a hedge for a read，后面就是具体的处理动作。
* Fail Fast的基本思路是将一些慢的server当作是故障了，然后按照处理故障的流程来处理。基本的方式是在pipeline packets写入的时候，记录packets写入的latencies。确定那些servers是slow通过判断一个server的请求延迟同样的一个PCT值最快的和最慢的sevrer时间的一个latency的一个倍数。这里采用PCT值为PCT95，倍数设置为3x。在作出那个server是slow的判断之后，client会使用一个cost function是否能从fail fast中获益。比如一个请求已经接近完成的时候发现是slow的server的话，可能就被认为fail fast是划不来的，而请求刚刚开始就发现为slow server的话，fail fast可能能够获益。
* Smart hedging for reads的思路是发现目前请求的延迟大于一个阈值的时候，发出一个hedge request，即在发现一个读请求的时候发现其延迟过高，就发送一个hedge请求来访问来的副本。这个问题常用的一种思路是设置一个固定的阈值，不过这种固定阈值的方式存在一些问题。这里采用的方式使用 per-client dynamic threshold：client会记录read请求的latencies，开始的时候使用一个固定的阈值来触发hedge请求。其判断一个server为slow的判断方式和写入的情况下判断的方式一样。在发出一个hedge请求之前，其判断后面采用的副本的处理情况，必要的时候采用指数回退和切换到另外的副本的方式。Smart hedging for reads会有一个retry policy来控制发送hedge请求的频率。
* 处理这些方式之外，eventually-consistent writes也是一种减少tail latency的方式，这种方式并发写所有的副本，在一部分副本ack之后就认为写入成功。这里考虑到这个对read的流程影响比较大，没有采用。根据之前的一些分析，这种方式对tail latency的优化还是比较明显的。Fail fast without replacement的思路则是在fail fast的基础上，不是使用改进请求到另外的节点而是直接移除slowest的server，然后缺失的副本在后台不齐。这个可以看作是fail fast和eventually-consistent writes的一种结合。但是其会影响到durability。

## MittOS: Supporting Millisecond Tail Tolerance with Fast Rejecting SLO-Aware OS Interface

### 0x20 基本内容

 MittOS这篇是在OS Kernel方面的一些改进Tail Tolerance，其核心思路是改变目前kernel  try best effects的工作方式，而是在估计用户提供的一个IO SLO，如果目前的状态已经无法实现这样的SLO了，就直接返回一个EBUSY的错误，而不是继续处理请求。这样的思路比较有意思，目前的一些环境中，在目的请求已经超过了系统处理能力的情况下，继续接受请求不是一个很好的做法，直接拒绝更好。拒绝请求以方便请求方改变请求目标然后重试请求。Paper中讨论了对Kernel的IO Scheduler、SSD的固件以及kernel的Cache等一些层面添加这样了Fast Rejecting SLO-Aware功能。MittOS在原来的基础上，可以添加一个关于SLO的参数，比如对于read syscall，`SLO = 20ms; ret = read(..,SLO); if (ret == EBUSY) { ...}`。添加这样的接口实现Fast rejection (“busy is error”)，并尽可能保留现有接口的一些内容(Keep existing OS policies)。

* 对于IO Scheduler，Noop Scheduler是一个比较简单的调度器，IO请求加入到一个队列里面，被FIFO的方式处理。这里通过队列计算需要的IO等待时间和，如果超过了deadline加上一个hop时间，则返回EBUS，即`T-wait>T-deadline+T-hop`。这里的一个主要问题就是如何计算处理一个iO请求需要的时间，这个时间通过这个请求的大小，请求的位置(offset)，以及目前IO的位置(offset)，加在device queue中的IO请求计算而来。另外每次会根据完成IO的时间来调整之前预计需要等待时间的计算。这里实际上计算的是一个next available time有没有超过这个请求的deadline。
* 对于CFQ Scheduler，这个要复杂一些。CFQ中存在gropu的概念，每个group中有各自有RealTime/BestEffort/Idle三种不同的services tree，不同services tree中的IO请求有会有各自的IO优先级。CFQ会按照RealTime/BestEffort/Idle的顺序处理，每种处理的时候讲一些请求添加到 FIFO dispatch queue中，后面乎进入device queue被处理。MittOS对CFQ的基础上，每个IO请求达到的时候，记录下请求属于哪一个group，哪一个servers tree以及其IO优先级。IO等待时间的计算通过对devices中depending的请求，dispatch queues中的请求，以及其它CFQ queue优先级更高的请求的时间和。
* 和前面的IO调度一样，在SSD Management发明的改进，MITTSSD，也是计算next available time有没有超过deadline。由于SSD里面会有多个channel、chip，这里会通过host-managed/software-defined flash这样的方式来获取channel/chip的一些信息。这里需要每个chip的next available time，MITTSSD通过计算SSD的性能来获取读写的延迟，以及通过T-chipNextFree+= 100μs 计算chip的下个空闲时间，然后通过T-wait =T-now −  T-chipNextFree+(60μs×#IOSameChannel)计算等待时间。写入的时候，还需要考虑有erase操作的情况。
* 另外一个是MITTCACHE，对于存在page cache的情况，read这样的请求计算in-memory read的情况。对于使用mmap这样读写文件的，由于其会变成直接读写内存，但是缺页也会造成IO。这里的方法是通过加入一个addr_check()，请求之前检查目前的IO情况。

## 参考

1. Tales of the Tail: Hardware, OS, and Application-level Sources of Tail Latency, SoCC '14.
2. Managing Tail Latency in Datacenter-Scale File Systems Under Production Constraints, EuroSys ’19.
3. MittOS: Supporting Millisecond Tail Tolerance with Fast Rejecting SLO-Aware OS Interface, SOSP '17.