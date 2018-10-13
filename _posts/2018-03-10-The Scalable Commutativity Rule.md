---
layout: page
title: The Scalable Commutativity Rule
tags: [Software Engineering]
excerpt_separator: <!--more-->
typora-root-url: ../
---



## The Scalable Commutativity Rule: Designing Scalable Software for Multicore Processors 



### 0x00 引言

   CPU的单核性能增长现在已经慢到不行，核心数量的增长还是很快的。如何改变软件让其适应越来越多的核心的CPU，这是个问题，233。

   API的设计(the design of the software interface )对系统可可拓展性有着很大的影响，一个例子就是open这个系统调用。POSIX标准规定返回的文件描述符必须是空闲的里面最小的，实际上对于这个API来说，文件描述符只要能保证是唯一的就行了，完全没必要有这样的要求。就是当初这样一个设定，导致了可拓展性的问题。在[MegaPipe](http://www.eecs.berkeley.edu/%7Esylvia/papers/osdi2012_megapipe.pdf)等之类的设计中都抛弃了这一个设定。

```
This forces the kernel to coordinate file descriptor allocation across all threads, even when many threads are opening files in parallel. This choice simplified the kernel interface during the early days of Unix, but it is now a burden on implementation scalability. It’s an unnecessary burden too: a simple change to allow open to return any available file descriptor would enable the kernel to choose file descriptors scalably.
```



### 0x01 A Rule for Interface Design 

​    这篇文章就给出了一种新的设计可拓展性的软件接口的方法，这个方法的核心就是:

```
  At the core of our approach is what we call the scalable commutativity rule: in any situation where several operations commute (meaning there’s no way to distinguish their execution order using the interface), there exists an implementation that is conflict free during those operations (meaning no core writes a cache line that was read or written by another core). Since conflict-free operations empirically scale, this implementation scales. Thus, more concisely, whenever interface operations commute, they can be implemented in a way that scales.
```

  在任何情况下，都存在一个实现可以使得所需要的操作是没有相互冲突的。很显然，操作指令没有冲突就以为着容易拓展。

  但是这个道理说起来谁都懂，可是能做到没有冲突就没有冲突吗？这个正是设计的难点所在，如何找到这样的一种设计。



### 0x02 SIM Commutativity  

   Paper中都一大段一大段抽象的分析证明，这里还是不要有这些东西了。这里将里面最核心的东西提炼出来。

  文中定义了一个可交换的原则，简而言之就是操作的顺序的可交换性，操作顺序和最终的结果直接的关系。最好的场景就是最终结果和操作的顺序无关。这种情况下实现可拓展是最可行的。设计这样的接口之前，我们要先知道不满足这个原则的地方在哪里。

   Paper设计实现了COMMUTER工具来分析接口之间的冲突情况。利用这个和另外一个叫做TESTGEN的工具，可以生存测试冲突覆盖率的测试用例。

```
To capture different conflict conditions as well as path conditions, we introduce a new notion called conflict coverage. Conflict coverage exercises all possible access patterns on shared data structures: looking up two distinct items from different operations, looking up the same item, and so forth.
```



### 0x03 实际例子

   文章中的例子是对于Linux的文件系统(这里的工作被发表在了SOSP 2017，时间怎么差了这么久？？).可以看到现在Linux的实现中存在很多的冲突。![scr-linux-fs](/assets/img/scr-linux-fs.png)

 而ScaleFS中，则消除了这些冲突，这样它就能实现更加好的可拓展新:

 ![scr-sv6-fs](/assets/img/scr-sv6-fs.png)



   感兴趣这是如何找到的吗。这里就需要2篇paper论文的时间，论文[3]，和论文[4]，[4]的结果发表在了SOSP2017上，基本的设计思想来自这篇论文，具体实用的基本方法来自论文[2]。后面会有关于这2篇论文的。



### 0x04 Guidelines

  继续将这篇很抽象的论文进行提炼，博客[5]是一个很好的参考。   

 4个设计commutative接口的指导:

* 分解复合的操作。一个简单的例子是fork syscall，它会导致很多额外的操作，要处理fork之后的内存写，地址空间操作以及文件操作等。这样就很容易导致操作的结果与顺序相关性就很强，不利于拓展。POSIX中posix_spawn和Windows中的CreateProcess类型，它直接创建一个新的process，就可以避免很多的问题。

* 允许非确定性的实现。就是上面提到的文件描述符的例子，每次返回空闲的最小的描述符这个设计就很糟糕。

* 准许弱的顺序保证。一个例子就是消息传递。当收到的消息的顺序必须要求和发出的顺序相同时，很不利于拓展。反之，没有这样的顺序要求是实现良好的可拓展性的空间就很大。

* 可以异步释放资源。POSIX中一些接口只有在全局生效的情况下操作才会返回，这对于接口使用的方便性来说很好。但是这样的操作可能会很昂贵。

  ```
  For example, writing to a pipe must deliver SIGPIPE immediately if there are no read FDs for that pipe, so pipe writes do not commute with the last close of a read FD. This requires aggressively tracking the number of read FDs; a relaxed specification that promised to eventually deliver the SIGPIPE would allow implementations to use more scalable read FD tracking.
  ```



 4个设计可拓展接口的设计模式:

* Layer scalability，分层可拓展性。使用容易拓展的方式，比如 linear arrays, radix trees, and hash tables 就比balanced trees 容易拓展。
* Defer work，推迟工作，场景的例子就是refcache，还有就是类似给予epoch的垃圾回收方式。
* Precede pessimism with optimism，先乐观后悲观。和OCC的思想比较相似。
* Don’t read unless necessary，不要读取不必要的东西。

>

### 0x05 总结

  这篇论文的感觉就是很抽象，文章长达47页，这里将字数控制在2000以内，当然发表在SOSP上的版本会短很多。重要的是其中的思想吧。

 >

## 参考

1. Austin T. Clements, M. Frans Kaashoek, Nickolai Zeldovich, Robert T. Morris, and Eddie Kohler. 2015. The scalable commutativity rule: Designing scalable software for multicore processors. ACM Trans. Comput. Syst. 32, 4, Article 10 (January 2015), 47 pages.  DOI: http://dx.doi.org/10.1145/2699681.
2. MegaPipe: A New Programming Interface for Scalable Network I/O, OSDI 2012.
3. OpLog: a library for scaling update-heavy data structures.
4. Scaling a file system to many cores using an operation log, SOSP 2017.
5. https://blog.acolyer.org/2015/04/24/the-scalable-commutativity-rule-designing-scalable-software-for-multicore-processors/，the morning paper.