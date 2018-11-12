---
layout: page
title: Log-Structured File System
tags: [File System]
excerpt_separator: <!--more-->
typora-root-url: ../
---



## Log-Structured File System

 

### 0x00 引言

  Log是Computer Science中一个非常重要的思想，与存储相关的地方都非常常见。Log-Structured File System(这里是LFS)是不同与常见的Unix FFS的一类文件系统，这篇论文发表于1992年(还是比我年龄要大呀)，比发表于1984年的Unix FFS晚了8年时间。计算机系统随着时间也发生了很多的变化，新的方法也会随之诞生。Log-Structured 是从文件系统，内存分配起到NVM空间管理，到Log-Structured 的数据结构，SSD的内部等等等，一大堆。可以来一个集合。



### 0x01 动机

 动机基于以下的观察:

1. System memories are growing;
2. There is a large gap between random I/O performance and se- quential I/O performance;
3. Existing file systems perform poorly on many common workloads;
4. File systems are not RAID-aware;



所以这里系统设计的核心就是[1]：

```
How can a file system transform all writes into sequential writes? For reads, this task is impossible, as the desired block to be read may be any- where on disk. For writes, however, the file system always has a choice, and it is exactly this choice we hope to exploit.
```

.

###  0x02 Writing To Disk Sequentially 

​    既然想写操作都变成顺序的，那么这里就顺序写好了。不管是什么类型的写操作，都顺序的写入磁盘，看看会产生哪些问题，如何在想办法解决这些问题，最后我就就得到了一个Log-Structured File System。



##### 问题

  如果我们更新一个文件的时候，文件的元数据也要走相应的修改。在Unix FFS之类的文件系统中，inode之类的元数据的位置是固定的，所以在更新了文件的数据之后，会直接找到对应的inode进行就地更新操作。但是对于LFS来说，这样的话写操作就不是顺序的了，会有对inode的随机写。

  这里的解决办法就是把inode也更新，顺序写到这里更新的操作之中，舍弃原来的旧的inode数据。

  看到起就想这样[1]:

```
+--------+----------+----------------------------------+
|        |          |  blk[0]:A0|                      |
|        |          |           |                      |
|        |          |    I      |                      |
|        |          |           |                      |
+--------+----------------------+----------------------+
         A0         |
         ^          |
         |          |
         +----------+
```

.

##### 问题

   上面的方案看上去是讲所有的写操作都顺序化了。但是来考虑一种case，就是当写入了一点数据(一个block)的时候，等待了一小段时间，写入另外的数据。这个时候会发现由于磁盘在旋转，原来写入的tail还要等待一段时间才能到达写入的位置，这样导致的问题就是写入的效率降低了。

​     解决方案就是write buffering，这个在文件系统中是很常用的方法了。LFS讲一次写入更新的chunk叫做segment。在前面的动机中提到，现在的系统内存越来越大，所以合理利用内存能有效地提升性能。多个在一起之后layout看上去就是这样：

![lfs-semgent](/assets/img/lfs-semgent.png)

### 0x02 The Inode Map  

   对于写来说，上面的方法看上去把写操作都顺序化了，当时当想要读的时候就发现了问题: 如果找到inode以继续后面的操作？

  使用一个间接层来解决这个问题，这里叫做Inode map，这个map使得我们可以用一个inode number就获取到这个inode最新版本的位置。

```
People often say that the solution to all problems in Computer Science is simply a level of indirection. This is clearly not true; it is just the solution to most problems (yes, this is still too strong of a comment, but you get the point). 
```

  这样的一个问题就是这个map必须是固定的吗？固定的话是不是又会导致非顺序写的问题。这里LFS使用的方法如inode的方法一样。

   问题还没有完，这样的话是不是进入了无穷的循环了。imap这么寻址？无论如何，FS中必须有一些东西是固定，要不然没法操作。LFS使用的方法是check- point region (CR)，CR保存了最新imap片段的指针。CR被周期性地更新，降低对系统性能的影响。

![lfs-cr](/assets/img/lfs-cr.png)



### 0x03 读操作

   到了这里来看看是如何读操作的:

1. 第一步是读取CR；
2. 然后LFS读取inode map到内存中(根据CR);
3. 根据一个inode读取文件到时候，先根据inode number在inode map中找到inode的位置信息；
4. 从磁盘中读取inode；
5. 利用inode的信息读取数据块。

### 0x04 目录

   到上面部分为止，基本的操作依据有雏形了。但这里还缺少了一个部分，文件系统是有目录结构的，如何处理目录。在Unix FFS中可以看到，创建一个文件也需要修改目录的数据。

  解决方案也是讲目录信息每次都更新写入，就想下面一样：

![lfs-dir](/assets/img/lfs-dir.png)



​    到了这里来更新一下0x03中读操作的过程:

1. 第一步是读取CR；
2. 然后LFS读取inode map到内存中(根据CR);
3. 根据一个inode读取文件到时候，先根据inode number在inode map中找到inode的位置信息；这里先读取目录的数据。
4. 从磁盘中读取inode；
5. 利用inode的信息读取数据块。
6. 在目录的数据中找到文件的inode number信息；
7. 重复3,4,5步骤，读取到最终的文件数据；

.

##### 问题

 这个问题是recursive update problem，当更新的时候，由于层层目录的关系，会导致目录都被更新。

 其实LFS在之前的内容中就已经把这个问题解决了，解决方法依然是inode map。inode的问题可能改变了，都是LFS的目录中不保存inode的位置信息，而是保存inode number。而一个文件更新之后inode位置可能变化了，但是inode number是不会变化的。	



### 0x05 Garbage Collection	

   当文件系统被更新的时候，写入的位置不断后移，之前的数据会不断的删除。LFS过期的数据会一直保存在原地，直到被垃圾回收的任务回收。

  前面提到了LFS中segment 的概念。LFS的GC也是根据segment 来进行的。LFS会周期性的读取segment 的信息当一个segment 里面存活的数据满足一定的条件，这里面存活的数据会被拷贝到新的地方，被删除的数据占用的空间也随之被回收。

  GC在Log Stuctured FS中是一个很重要的内容，可以参看原论文[2]和之后的一些优化的论文,比如[3]。



### 0x06 Crash Recovery And The Log 

  文件系统提高持久化的数据保存。我们前面看到，LFS大量的数据是cache在内存中的，这样就需要处理系统crash的问题(也不仅仅是这个原因)。LFS需要解决以下的问题：

1. CR原子更新的问题；
2. CR周期更新过程中CR数据丢失的问题。

  解决第一个问题的方法是使用2个CR，每次更新的时候是更新另外一个。先修改CR，然后在修改指针的方式。此外，CR在前后两端保存了时间戳，当是完整更新的时候这两个时间戳是系统的，不相同则认为没有更新成功。

  解决第二个问题使用了类似数据中的方法。从最后的检查点开始，查找有效的更新。如果存在，则相应更新操作。

.

### 0x07 Summary

  经典文章，值得一读。

```
some modern commercial file systems, including NetApp’s WAFL, Sun’s ZFS, and Linux btrfs, and even modern flash-based SSDs, adopt a similar copy-on-write approach to writing to disk, and thus the intellectual legacy of LFS lives on in these modern file systems. In particular, WAFL got around cleaning problems by turning them into a feature; by providing old versions of the file system via snapshots, users could access old files whenever they deleted current ones accidentally.
```

.

## 参考

1. “Operating Systems: Three Easy Pieces“ (Chapter: Log-structured File Systems) by Remzi Arpaci-Dusseau and Andrea Arpaci-Dusseau. Arpaci-Dusseau Books, 2014. 
2. The Design and Implementation of a Log-Structured File System, TOCS 1992.
3. Improving the Performance of Log-structured File Systems with Adaptive Methods, SOSP 1997.