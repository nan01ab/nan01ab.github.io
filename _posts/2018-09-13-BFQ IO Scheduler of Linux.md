---
layout: page
title: BFQ IO Scheduler of Linux
tags: [Operating System]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## BFQ I/O Scheduler of Linux

### 0x00 引言

   Linux目前使用的IO Scheduler有CFQ，NOOP和Deadline。在最近几年，Linux的Block Layer做了很多的有哈，最大的一个改变就是添加了multi-queue block layer。相适应的，IO Scheduler也出现了新的变化，在4.12版本中新加入了Budget Fair Queueing (BFQ) Storage-I/O Scheduler和Kyber multiqueue I/O scheduler。前者有不少的资料，这里先记录一下BFQ。Kyber multi-queue I/O scheduler没有文档资料，不过这个应该是一个比较简单的，代码只有约1000行，先研究研究再说。BFQ包含了很多细节的优化，这里以后慢慢完善。



### 0x01 BFQ基本思路

   BFQ保持每一个进程一个IO请求队列，不同于CFQ用RR的方式轮询这些队列的方法，BFQ给这些队列一个I/O budget，这个I/O budget代表了下次能写入到磁盘的扇区(这里是HHD上面的说法)的数量。这个I/O budget的计算时一个很复杂的过程，这里只会简单的说明，不会很具体的讨论。这个I/O budget主要和进程的行为有关。

![bfq-arch](/assets/img/bfq-arch.png)

一个基本的过程如下：

1. 挑选一下一个服务的进程，这里使用内部的B-WF2Q+，一个内部的fair-queueing调度器；
2. 请求分发，等待调用相关函数，处理请求，这时候的处理逻辑是：
   1. 一个内部的C-LOOK调度器选择这个应用的一个请求，从它的请求队列中那处理，然后存储设备处理请求。这个C-LOOK既能适应HHD也能适应SSD。
   2. 应用的budget会逐渐递减；
   3. 如何这个应用的请求处理完成，or 没有了budget之后，执行下面的第3步；
3. 这个应用会被退出服务，赋予一个新的budget，这里会有算法计算新的这个budget；

在BGQ中，每个进程都会赋予一个固定的权重，从总体上来说，这个进程使用的磁盘的吞吐量只会和这个权重相关，和上面的budget其实没有关系。也就是说，这个磁盘的带宽会被竞争使用这个磁盘的进程共同分配。这个看上去反直觉，因为上面提到了budget，这个数值大的能帮助进程获取更加多的资源。但是这里如果是budget更加大，B-WF2Q+会将其的服务推迟得更晚，更小的budget能更及时地得到服务。进程的这个budget的计算时和这个进程的权重时没有关系了，上面提到了，这个budget更多由进程的行为决定的。如果对CFS的进程调度器也熟悉的话，这里的很多思想也是类似。BFQ、CFQ和CFS的一个目标都是公平。此外，区分进程的特点，比如区分交互进程和非交互进程以提供更加好的服务。

#### Budget assignment 

  最为一个常识，要想获得最高的磁盘性能，访问最好都是顺序的，很显然这里如果是顺序访问磁盘的进程，最好就能赋予一个更加大的budget。BFQ会通过一系列的方法来计算磁盘的性能特点和一个进程访问的特点。通过 feedback-loop算法，BFQ尝试去计算下一次的budget，让这个budget尽可能的接近进程的需求，当然这里值的计算要考虑到目前的系统状况和磁盘的特点。

  BGQ会给一个进程最大的使用时间，这样可以避免随机访问的进程长时间的占有磁盘而降低了吞吐量(这里可以猜想，这里是因为随机访问的应用实际写入的数据比较小，磁盘随机访问的性能远低于顺序访问的性能，而导致budget消耗速度比较慢，从而导致了一直占有磁盘)。

```
To further limit the extent at which random applications may decrease the throughput, on a budget timeout BFQ also (over)charges the just deactivated application an entire budget even if the application has used only part of it. This reduces the frequency at which applications incurring budget timeouts access the disk.
```

.

### 一些细节和优化

TODO

## 参考 

1. https://lwn.net/Articles/720675/, Two new block I/O schedulers for 4.12.
2. https://lwn.net/Articles/601799/, The BFQ I/O scheduler.
3. Improving Application Responsiveness with the BFQ Disk I/O Scheduler, SYSTOR ’12, https://core.ac.uk/download/pdf/54000085.pdf
4. http://algo.ing.unimo.it/people/paolo/disk_sched/mst-2015.pdf, Evolution of the BFQ Storage-I/O Scheduler 