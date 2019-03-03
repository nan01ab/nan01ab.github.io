---
layout: page
title: The Tail at Scale
tags: [Distributed]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## The Tail at Scale

### 引言

  这个大名鼎鼎的 JEFFREY DEAN发表的一片关于**tail-tolerance**的一篇文章。一般都知道接口的一个重要的衡量指标就是TOP 90, TOP 99之类的。常见的互联网的服务比如网页加载速度、相关接口的反应时间等等是直接影响到用户体验的。Amazon和Google都要相关的数据表明加载时间长多少ms，用户就会损失多少。这个可以在网上找到相关的文章。



### Tail-Tolerance

   在较复杂的系统中，长尾延时总是会存在。造成这个的原因非常多，常见的有网络抖动，GC，系统调度等。文章中有总结如下:

* Shared resources.  公用资源产生的相互影响；
* Daemons.
* Global resource sharing. 比如交换机，共享的文件系统；
* Maintenance activities. 
* Queueing.  
* Power limits. 
* Garbage collection. 这里不仅仅包括类似JVM的GC，还有SSD这类hardware的内部行为；
* Energy management. 

key insights：

```
even rare performance hiccups affect a significant fraction of all requests in large-scale distributed systems.

eliminating all sources of latency variability in large-scale systems is impractical, especially in shared environments.

using an approach analogous to fault-tolerant computing, tail-tolerant software techniques form a predictable whole out of less-predictable parts.
```

##### 为什么要关注这个

  假设一个系统的一般响应时间是10ms，由于上面的各种这样的元素，导致了系统的TOP 99为1s。看上去平均数据很棒，但是对于现在常见的互联网服务来说，一个用户的请求上去，时间上会调用后面的很多的服务，假设每一个服务的性能指标都想这个一样，如果它实际上引发了100个服务的调用了，就会有超过63%的用户的响应时间超过1s。

```
 If a user request must collect responses from 100 such servers in parallel, then 63% of user requests will take more than one second. Even for services with only one in 10,000 requests experiencing more than one-second latencies at the single-server level, a service with 2,000 such servers will see almost one in five user requests taking more than one second.
```

  更加可怕的是，即使TOP 99.99的影响时间都很好，各个服务的tail latency加在一起也会把这个响应时间明显放大.

```
The 99th-percentile latency for a single random request to finish, measured at the root, is 10ms. However, the 99th-percentile latency for all requests to finish is 140ms, and the 99th-percentile latency for 95% of the requests finishing is 70ms, meaning that waiting for the slowest 5% of the requests to complete is responsible for half of the total 99%-percentile latency. 
```

  用概率论的来说这也很容易理解，虽然发生一次的概率比较低，但是重复多次要求它一次都没有发生的概率也是比较低的。

###  Design  Patterns to Enhance Tail Tolerance

  这个问题理解起来还是很容易的。重点在与如何解决呢？这个命题太广了，就只看看文章中给了哪些解决这些问题的模式。

* Hedged requests. 简单的说就是不只请求一个，而是同时请求多个，等最快的一个返回就可以了。这个方法简单粗暴，工作中也使用过。缺点就是会消耗很多额外的资源，而且不是使用的服务(or 接口)都能这么用的。

  ```
   For example, in a Google benchmark that reads the values for 1,000 keys stored in a BigTable table distributed across 100 different servers, sending a hedging request after a 10ms delay reduces the 99.9th-percentile latency for retrieving all 1,000 values from 1,800ms to 74ms while sending just 2% more requests.
  ```

* Tied requests. 为了解决排队导致的tail latency，这里使用的方式是给多个服务器发送请求并排队在这个服务器的队列里面，服务器之间可以更改对应任务拷贝的状态。简而言之，就是一些服务器都安排任务，谁先做谁做，还可以告诉别人不要做了。

  ```
   In it, sending a tied request that does cross-server cancellation to another file system replica following 1ms reduces median latency by 16% and is increasingly effective along the tail of the latency distribution, achieving nearly 40% reduction at the 99.9th-percentile latency. The second scenario is like the first except there is also a large, concurrent sorting job running on the same cluster contending for the same disk resources in the shared file system. 
  ```

* Micro-partitions. 这样是解决一些负载不均衡的问题，通过使用更多比服务中的机器要多的的分区，然后在这些机器之间移动分区来解决不平衡的问题(看起来比较抽象)。文章中还是bigtable的一个例子：

  ```
  With an average of, say, 20 partitions per machine, the system can shed load in roughly 5% increments and in 1/20th the time it would take if the sys- tem simply had a one-to-one mapping of partitions to machines.
  The BigTable distributed-storage system stores data in tablets, with each machine managing between 20 and 1,000 tablets at a time. Failure-recovery speed is also improved through micro-partitioning, since many machines pick up one unit of work when a machine failure occurs.
  ```

* Selective replication. 可以看作是Micro-partitions完善版本，简而言之就是增加热点分区的副本的数量:

  ```
  Google’s main Web search system uses this approach, making additional copies of popular and important documents in multiple micro-partitions. At various times in Google’s Web search system’s evolution, it has also created micro-partitions biased toward particular document languages and adjusted replication of these micro-partitions as the mix of query languages changes through the course of a typical day.
  ```

* Latency-induced probation. 这个方法和熔断的方法比较相似，就是发现了反应慢的机器之后，可以考虑应该是它or它依赖的某些东西出来异常，就可以暂时把它从服务的机器中剔除，然后测试什么时候恢复正常在重新加入.

  ```
   The source of the slowness is frequently temporary phenomena like interference from unrelated network- ing traffic or a spike in CPU activity for another job on the machine, and the slowness tends to be noticed when the system is under greater load. However, the system continues to issue shadow requests to these excluded servers, col- lecting statistics on their latency so they can be reincorporated into the service when the problem abates. 
  ```

* Good enough. 一些情况下，是可以接受那么有点儿不完整的结果的，接受一点不完整有时候能带来latency的降低，提高体验:

  ```
  In large IR systems, once a sufficient fraction of all the leaf servers has responded, the user may be best served by being given slightly incomplete (“good-enough”) results in exchange for better end-to-end latency.
  ```

* Canary requests. Canary是金丝雀的单词，这里引用的是以前的矿工经常用它来探测有毒气体。这里的含义就是先请求一部分试试看:

  ```
  To prevent such correlated crash scenarios, some of Google’s IR systems employ a technique called “canary requests”; rather than initially send a request to thousands of leaf servers, a root server sends it first to one or two leaf servers. The remaining servers are only queried if the root gets a successful response from the canary in a reasonable period of time. 
  ```

上面这些方法都有各自的适应环境(一句废话)。

###  参考

1. The Tail at Scale, Software techniques that tolerate latency variability are vital to building responsive large-scale Web services, BY JEFFREY DEAN AND LUIZ ANDRÉ BARROSO 
2. https://blog.acolyer.org/2015/01/15/the-tail-at-scale/, the morning paper.