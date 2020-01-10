---
layout: page
title: Soft Updates -- A Solution to the Metadata Update Problem in File Systems
tags: [File System]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## Soft Updates: A Solution to the Metadata Update Problem in File Systems
### 0x00 引言

  这篇Paper是讨论解决文件系统中一致性问题。关于文件系统中一致性的解决方法主要有Journaling、CoW的解决方式。另外这篇Paper提出的Soft Updates也是一种方法。但是在实际的使用中，Soft Updates并没有得到多少实际地使用，主要是因为其实现的难度很大。一般文件系统使用的还是日志或者是写时复制的技术。

### 0x01 基本思路

  Soft Updates的核心思路是满足更新之间的依赖关系。这些操作的顺序的限制总结起来主要就是3点，

*  不能指向一个没有初始化的结构，比如一个目录指向的inode必须已经初始化了；
* 在之前指向一个资源的指针没失效之前不能重新使用，比如必须保证一个inode指向的一个数据块失效之后，这个数据块才能被重新分配；
* 对于一个存活的资源，在新的指针没有设置之前不能重置之前指向这个存活资源的指针，比如重命名一个文件的时候，在一个新的文件名没有写入的时候不要去移除旧的文件名；

  满足这样的要求下，文件系统可以保证正确性，在Crash之后一定会是可以恢复的，但是可以导致磁盘空间“泄露”，这个通过额外的方法解决。Soft Updates通过追踪更新操作之间的依赖关系，来安排写入的顺序来保证一致性。Soft Updates在开始的时候准备使用DAG(有向有环图)来表示这些操作之间的依赖关系(在这些文件系统的操作之中，是很容易造成循环依赖的关系的)。但是在实际考虑之后发现这种方式操作的难度太大，转而使用了另外的方式。在Soft Updates中，依赖的关系是在很小的粒度上面保持的(在字段or指针基本的粒度)。对于每一个更新操作，都会保存之前和之后的版本，以及它依赖的更新操作的链表。

<img src="/assets/img/supdates-cyclic.png" alt="supdates-cyclic" style="zoom:67%;" />

  上面的图是一个循环依赖的例子。系统在一个目录下面创建文件A之后，会导致一个目录块到Inode块的依赖关系，前者依赖于后者，所以后者必须在前者之前持久化。之后的操作在该目录中删除了文件B，导致了一个Inode块到目录块的依赖关系。这样就形成了一个循环依赖的关系。这里Soft Upfates解决的方法是回滚操作，Soft Upfates写入的顺序是可以随意的。。加入这里要写入目录的数据块，由于存在被依赖者Inode的数据库，所以不能写入。这里就会暂时地回滚创建A的操作，就可以将其写入，之后在重新操作即可。这里实际上会导致目录块写入两次。

<img src="/assets/img/supdates-redo.png" alt="supdates-redo" style="zoom: 67%;" />

  上面的图显示了这样的操作过程。在内存中两个操作完成之后，形成了一个循环依赖的关系。为了写入目录块，写入的是删除文件B之后的目录块，内存中的目录块依赖于Inode块的信息还保留。下一次的操作就是将初始化的A的Inode的Inode块写入到磁盘，满足前面的3点的要求。最后再是写入了添加文件A的目录块更新，所有的依赖关系解除，操作完成。可以发现在这个过程中目录块写入了2次，在中间任何状态Crash的时候，文件系统是安全的，但是可能导致Inode的泄露。

  Sofe Updates实现在4.4BSD的FFS中。可以发现，要追踪这类细粒度的依赖关系，实际上是与文件系统实现是强依赖关系的，也就是说Soft Updates实现的可移植性很低。在基于FFS的实现中，主要有4个主要的结构修改需要顺序的更新：1. 块分配，2. 块回收，3. 增加链接(比如创建文件)，4. 移除链接。其操作的实现主要就是要满足前面的几个限制条件。具体的信息比较多[1]。

### 0x03 恢复 和 文件系统语义

  在基于Soft Updates实现的文件系统中，在系统Crash之后恢复的时候可以马上挂起使用。文件系统也可能存在以下的一些不一致的状态，

* 没有使用的Blocks没有出现在空闲空间中；
* 没有使用的inode被标记为已经使用了；
* Inode上面的引用计数可以多于实际的值，这个可以导致资源不能及时得到回收。

 系统可以通过在后台运行fsck这样的工具来处理这样的情况，

```
Maintaining the dependencies described is sufficient to guarantee that the on-disk copies of inodes, directories, indirect blocks, and free space/inode bitmaps are always safe for immediate use after a system failure. However, FFS maintains a number of free block/inode counts in addition to the bitmaps. These counts are used to improve efficiency during allocation and therefore must be consistent with the bitmaps for safe operation. 
```

对于文件系统语义，Soft Updates并没有多少改变，和一般的FFS的实现基本系统。

### 0x04 评估

  这里的具体信息可以参看[1]，由于这篇Paper差不多是20年前的原因，里面测试使用的硬件在今天看来是古董了2333，

<img src="/assets/img/supdates-perf.png" alt="supdates-perf" style="zoom:67%;" />

## 参考

1. Soft Updates: A Solution to the Metadata Update Problem in File Systems, TOCS 2000.