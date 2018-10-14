---
layout: page
title: Two Scheduling Methods
tags: [Scheduling]
excerpt_separator: <!--more-->
typora-root-url: ../
---



## Two Scheduling Methods 



### 0x00 引言

  MLFQ应该OS课程都会将到的一个Scheduling算法吧。因为某些原因需要复习一下几年前学过的内容。

```
The Multi-level Feedback Queue (MLFQ) scheduler was first described by Corbato et al. in 1962 in a system known as the Compatible Time-Sharing System (CTSS), and this work, along with later work on Multics, led the ACM to award Corbato its highest honor, the Turing Award. The scheduler has subsequently been refined throughout the years to the implementations you will encounter in some modern systems.
```

.

### 0x01 How To Change Priority

   算法的核心就是到达一下的目标：

```
How can we design a scheduler that both minimizes response time for interactive jobs while also minimizing turnaround time without a priori knowledge of job length?
```

  书中这篇讲述的方式和《Paxos Made Simple》的一样，都是先提出一个问题or目标，给出基本的方案。然后分析基本的方案存在的问题，之后解决这些问题，得到一个达到目标的解决方案。

  Scheduler 一般都存在一些基本的规则，先来看看这里的两个基本的规则:

```
• Rule 1: If Priority(A) > Priority(B), A runs (B doesn’t).

• Rule 2: If Priority(A) = Priority(B), A & B run in RR.
```

.   

   要改变Priority的一个原因就是根据任务运行的状况观测太是出于哪一种类型的人物，对于交互式的任务(or IO密集型的)，一般的做法就是提高它的Priority，来达到更好的反应时间。交互式任务的一个特点就是等待(想想一个人键盘的输入，从按下空格键到按下回车键对于计算机来说都是一段很长的时间了，对于计算机来说，它等你按回车键等了很久)。所以这里一个基本的思路就是，对于每次运行丢用不完自己时间片的任务，就保持它的Priority，对于能用完自己时间片的任务，更加可能是非交互式的任务，所以就降低它的Priority：

```
• Rule 3: When a job enters the system, it is placed at the highest priority (the topmost queue).

• Rule 4a: If a job uses up an entire time slice while running, its pri- ority is reduced (i.e., it moves down one queue).

• Rule 4b: If a job gives up the CPU before the time slice is up, it stays at the same priority level.
```

.

##### 存在的问题

1. 饥饿，交互式的任务一多，其它类型的任务就很难得到运行的机会；
2. 不安全，应用可以hack这个sheduler，通过在这个时间片的末尾发出IO请求，它就能一直保持住它的优先级；
3. 任务的行为不是一成不变的，一个任务已开始可能是IO密集型的，之后会变成CPU密集型的，反之亦然。然而在这里一旦开始被认为是CPU密集型的，它就没机会变为IO密集型的了(sheduler的看法)



### The Priority Boost 

   为了解决之前存在的问题，引入了规则5:

```
• Rule 5: After some time period S, move all the jobs in the system to the topmost queue.
```

  很显然的上面的第1，3个问题就被解决了。



### Better Accounting 

​    上面提到的第2个问题还是没有解决。之所以存在这个问题，是因为上面的rule 4a,4b。那就将rule修改一下:

```
Rule 4: Once a job uses up its time allotment at a given level (regardless of how many times it has given up the CPU), its priority is reduced (i.e., it moves down one queue).
```

  现在，只要它用完了在一个allotment上面的时间片，它就会被削减priority。对于交互型的任务，它就越可能留在更高的优先级上，对于非交互型的任务，它就会越快地滑向低优先级。



### Summary 

  当然实际上的相同会比这个复杂很多，不过这里基本思路都有了。实际使用的系统中，有Solaris和FreeBSD使用了这个类型的Scheduler。之前Linux 使用的O(1)也是类型的。

```
MLFQ is interesting for the following reason: instead of demanding a priori knowledge of the nature of a job, it observes the execution of a job and prioritizes it accordingly. In this way, it manages to achieve the best of both worlds: it can deliver excellent overall performance (similar to SJF/STCF) for short-running interactive jobs, and is fair and makes progress for long-running CPU-intensive workloads. For this reason, many systems, including BSD UNIX derivatives, Solaris, and Windows NT and subsequent Windows operating systems use a form of MLFQ as their base scheduler.
```



  总结一下：

```
• Rule 1: 如果A的优先级 > B的优先级, A运行.

• Rule 2: 如果A的优先级 == B的优先级, A B以RR的方式运行.

• Rule 3: 任务最开始加入的时候，都将它运行在最高的优先级.

• Rule 4: 在一个队列上面运行完时间片之后，将削减优先级，降低到更加低的队列.

• Rule 5: 周期性地将所有任务提升到最后优先级.
```

.

### 0x02 Proportional Share 

  除了上面说的Multi-level Feedback Queue之外，proportional-share也是一种设计思想。书中以lottery scheduling调度方法为例子，对于这种思路来说，核心就是:

```
How can we design a scheduler to share the CPU in a proportional manner? What are the key mechanisms for doing so? How effective are they?
```

.

#### 最基本的思路

  lottery scheduling使用的一个最基本的概念就是：tickets，凭证。使用tickets来代表了使用分享到的资源，tickets的比例代表了分享到的资源的比例。它使用一种概率性的方法来决定下一个运行的进程，比如下面这个例子，系统必须知道这里一共有多少个tickets(这里就是100)，然后挑选一个随机的ticket，数字是0-99。如果是0-74则运行A，74-99则运行B。

```
Let’s look at an example. Imagine two processes, A and B, and further that A has 75 tickets while B has only 25. Thus, what we would like is for A to receive 75% of the CPU and B the remaining 25%.
```

.

#### Ticket机制

   lottery scheduling也提供了一些方式来操作tickets。一个概念就是ticket currency，currency使得用户可以决定以怎么样的比例来运行自己的jobs，系统会自动将这个currency转化为全局的tickets。比如用户A给自己的两个jobs每个500tickets，B用户给自己的一个job 10tickets，则对应到全局的rickets就是A1 50，A2 50，B1 100。这个有点类似于Linux里面的组调度。

  另外的一个概念就是ticket transfer，它使得进程之间可以临时地将自己的tickets转移到另外一个进程，这个在client-server的模式下特别有用。在client给server发送请求之后，client为了加速server的运行，可以将ticket暂时转移。还有一个概念就是 ticket inflation，这个使得进程可以暂时提高自己的tickets.

```
Rather, *inflation* can be applied in an environment where a group of processes trust one another; in such a case, if any one process knows it needs more CPU time, it can boost its ticket value as a way to reflect that need to the system, all without communicating with any other processes.
```

.

#### The Linux Completely Fair Scheduler (CFS) 

 目前linux使用的默认的调度器CFS的思路就和fair-share scheduling 很相似。CFS的基本思路就是尽可能的将CPU平均分配给所有的进程，它使用一个基于计数的机制就是virtual runtime (vruntime)。进程运行的时候就会累积vtime，在一般情况下，进程vtime增加的速度是一样的，与物理上的时间成一定的比例(具体参考更加详细的资料)。

.

  没有调度器是完美的， fair-share schedulers 存在的问题一是这样的方法与IO不怎么协调，频繁使用IO的进程可能得不到它们分到的CPU份额。另外一个问题就是如何赋予进程合适的ticket或者叫优先级。





## 参考

1. “Operating Systems: Three Easy Pieces“ (Chapter: The Multi-Level Feedback Queue) by Remzi Arpaci-Dusseau and Andrea Arpaci-Dusseau. Arpaci-Dusseau Books, 2014. 
2. “Operating Systems: Three Easy Pieces“ (Chapter: Scheduling: Proportional Share) by Remzi Arpaci-Dusseau and Andrea Arpaci-Dusseau. Arpaci-Dusseau Books, 2014. 

