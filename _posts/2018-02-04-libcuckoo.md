---
layout: page
title: Algorithmic Improvements for Fast Concurrent Cuckoo Hashing
tags: [Data Structure, Key-Value]
excerpt_separator: <!--more-->
typora-root-url: ../
---



## Algorithmic Improvements for Fast Concurrent Cuckoo Hashing 



### 0x00 引言

  这篇paper是MemC3[2]的进一步的工作，发表的时间晚了一年(2014，想想当时年轻的我，心塞塞的)。这里继续讨论了对hash table的一些优化，主要是对MemC3中的hash table的设计，特别是并发方面。此外，还利用了HTM做的一些优化。



### 0x01  Improve Concurrency 

   先来看一看结果，性能数据还是灰常赞的(可怜的TBB，经常被比较，总是被打败).

![fcch-throughput](/assets/img/fcch-throughput.png)



 并发方面的优化基于一下基本principle:

```
P1: Avoid unnecessary or unintentional access to common data.  这个基本不用说了，最基本的套路.

P2: Minimize the size and execution time of critical sections. 同上.

P3: Optimize the concurrency control mechanism. 这里的名堂就多了去了。这里还讨论了使用HTM的优化。
```

.

### 0x02 Algorithmic Optimizations 



##### Lock After Discovering a Cuckoo Path 

  这个优化是为了减少临界区的大小，这里给出paper中算法的描述就很好理解了:

![fcch-lock-after](/assets/img/fcch-lock-after.png)

​	

  原算法是先查找是否还有空闲的slot，如果有就添加，没有就找出一个path在添加，这样的话主要的过程是一直被lock的。优化的方式如第二个算法：将lock拆成了几个部分，其中一个最主要的是查找path的过程是不在lock的范围内的。这样的优点就是比较耗时间的操作在临界区之外，缺点是执行根据path的isnert操作时，需要额外的检查。



##### Breadth-first Search for an Empty Slot 

   原来采用的查找空slot的方法是贪婪算法，这里优化为BFS算法。原算法的一个优化是同时查找多个可行的path(实际上就是2个)，本质上是类似DFS的算法。BFS的优化可以理解为查找了所有的可行的path，来找到一个更加好的。

```
This optimization is key to reducing the size of the critical section: While the total number of slots examined is still M, this is work that can be performed without a lock held. With BFS, however, at most five buckets must be examined and modified with the lock actually held, reducing both the duration of the critical section and the number of cache lines dirtied while doing so.
```

.

##### Increase Set-associativity

 更高的组相联程度有以下方面的影响:

1. 会降低查询的性能，需要查找的slot更加多了。而且数据变得更加分散，缓存的友好度降低；
2. 会提好写的性能，因为有空slot的可能性提高了。

 个人觉得这里更多的是权衡，trade-off.

.

#####  Fine-grained Locking 

   经过一番优化之后，这里的临界区都比较小了，采用spin lock是一个更加好的选择。

```
Here we favor spinlocks using compare-and-swap over more general purpose mutexes. A spinlock wastes CPU cycles spinning on the lock while other writers are active, but has low overhead, particularly for uncontended access. Because the operations that our hash tables support are all very short and have low contention, very simple spinlocks are often the best choice.
```

.

### 0x02 HTM

   这里讨论了使用Intel TSX作为lock的一个替代。TSX存在的一个问题是在一些情况下会造成transaction的abort(事务内存)，主要是以下的原因:

```
1. Data conflicts on transactionally accessed addresses;

2. Limited resources for transactional stores;

3. Use of TSX-unfriendly instructions or system calls. such as brk, futex, or mmap. 
```

  前面的两个原因通过之前提到的优化就已经被很好缓解了。第三个原因就是在实现中尽量避免了使用这些syscalls。



![fcch-tsx](/assets/img/fcch-tsx.png)





 具体地评估数据可以参看原论文[1]. 此外，这个的代码已经来源了在github上: libcuckoo.



## 参考

1. Algorithmic Improvements for Fast Concurrent Cuckoo Hashing， EuroSys 2014.
2. MemC3: Compact and Concurrent MemCache with Dumber Caching and Smarter Hashing，NSDI ’13.