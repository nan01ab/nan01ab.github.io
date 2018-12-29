---
ilayout: page
title: Rethink the Sync
tags: [File System]
excerpt_separator: <!--more-->
typora-root-url: ../
---



## Rethink the Sync



### 0x00 引言

  这篇Paper讲的是同步IO与异步IO的一些问题，Paper中提出了一种`external synchrony`的模型，可以帮助文件IO在保持同步IO的可靠性和简便性的同时实现异步IO般的性能。

```
For I/O-intensive benchmarks such as Postmark and an Andrew-style build, the performance of xsyncfs is within 7% of the default asynchronous implementation of ext3. ... Xsyncfs is as much as an order of magnitude faster than the default version of ext3 mounted synchronously, which allows data to be lost on power failure because committed data may reside in the volatile hard drive cache. Xsyncfs is as much as two orders of magnitude faster than a version of ext3 that guards against losing data on power failure.
```

.

### 0x01 基本思路

   Paper引入了同步操作的不同的定义，external synchrony的基本概念不同于传统的同步IO的定义。

* 在传统的同步IO的定义中，同步的概念为IO操作会一致阻塞到IO完成才会返回，而异步操作就是付出IO请求直接返回，不会等待IO操作的实际完成。而这里的定义的externally synchronous I/O则是从可见性的角度出发来定义，它的同步定义为这个IO操作返回的时候它就会是可见的，也就是会被之后的操作感受到(而异步操作不提供这个包装，因为这个操作可能根本没有开始执行)，另外的一个特点和异步的操作系统，它不会保证IO操作在返回的时候就已经完成了，这里其实就可以理解为这个操作可能还在OS掌握中并没有具体去执行，但是当OS处理与这个(些)IO操作相关的请求时，可以保证这个(些)的操作结果被后面的操作感受到。从这个角度来看，externally synchronous I/O不像是不同IO的一种优化，而是异步IO的一个改进；

*  另外，externally synchronous I/O是以用户为中心的视角。而传统的是以应用为中心的视角。在系统的情况下，系统被分为kernel和user两个层次(部分)，前者被认为为内部状态，而后者被视为外部的状态。而在externally synchronous I/O的视角下，系统被分为三个层次：1. kernel和应用，它们一同被视为内部状态，2. 外部的interfaces，被视为外部的可见性。在这样的视角下，一个改变应用状态的操作，比如从syscall返回，不会被视为一个外部可见的事件；

* 这这样定义下，下面的图表示了简单情况下面的示例。为了保证准确性，操作这件的依赖关系和先后的顺序必须有系统保持，

  ```
  An operating system that supports external synchrony must ensure that external output occurs in the same causal order in which it would have oc- curred had I/O been performed synchronously. Specifically, if an external output causally follows an externally synchronous file I/O, then that output cannot be observed before the file I/O has been committed to disk.
  ```

![esync-concept](/assets/img/esync-concept.png)

* 通过上面提到的一些操作，externally synchronous I/O就有可能实现和异步IO相似的性能，同时拥有一些同步IO的一些特点。IO到最后还是得最终提交的，在这样的情况下面，系统可以通过组提交的方式来提高性能。另外的一个考虑的点就是什么时候应该执行提交。一个基本的方法就是定时提交的方式，另外这里一个基本的权衡就是吞吐和延时之间的权衡。Paper 中提出的方法叫做output-triggered commits，

  ```
  Latency is unimportant if no external entity is observing the result. Specifically, until some output is generated that causally depends on a file system transaction, committing the transaction does not change the observable behavior of the system. Thus, the operating system can improve throughput by delaying a commit until some output that depends on the transaction is buffered.
  ```

* Externally synchronous I/O也存在一些缺点，一个最致命的驱动就是在IO操作返回之后数据并没有被实际保存而这个时候应用认为IO操作已经完成，就会继续执行之后的操作，这个致命的缺陷也就是导致来它几乎不会被真实的使用。这里是不是可以用Log的方式解决部分问题，还有在非易失性内存的情况下有上面好办法；



### 0x02 实现

  对于externally synchronous I/O的实现，需要做的主要是两件事情：1. 操作之间的依赖关系；2. 缓冲这些操作。

* OS Support，对于操作系统的改动主要就是追踪操作之间的依赖情况，已经操作的提交情况等，

  ```
  ... two new data structures to the kernel. A speculation object tracks all process and kernel state that depend on the success or failure of a speculative operation. Each speculative object in the kernel has an undo log that contains the state needed to undo speculative modifications to that object. 
  ```

* FS Support，这里的主要要修改的就是保持操作的顺序来。另外在现在的IO栈中，操作都是“乱序”执行了，一个最明显的地方就是在Block Layer会对操作进行重新排序，以求获取更好的性能。这里必须保持操作被具体执行到存储设备上面的顺序，否则就可能导致错误。
* Xsyncfs & Output-Triggered Commits。Paper中的实现以ext3文件系统为基础。这里不具体看在ext3上面实现的细节了，[1].



### 0x03 评估

 ![esync-perf](/assets/img/esync-perf.png)



## 参考

1. Nightingale, E. B., Kaushik, V., Chen, P. M., and Flinn, J. 2008. Rethink the Sync. ACM Trans. Comput. Syst. 26, 3, Article 6 (September 2008), 26 pages. DOI=10.1145/1394441.1394442. http://doi.acm.org/10.1145/1394441.1394442

