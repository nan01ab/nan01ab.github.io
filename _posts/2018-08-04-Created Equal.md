---
layout: page
title: All File Systems Are Not Created Equal
tags: [File System]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## All File Systems Are Not Created Equal: On the Complexity of Crafting Crash-Consistent Applications

### 0x00 引言

  这篇Paper是一篇关于Crash-Consistency的Paper。对于如何写出正确的文件操作的代码很有价值。Paper中提出了一个BOB的工具来测试文件系统的persistence properties，开发了一个ALICE的工具来分析应用代码的crash vulnerabilities。



### 0x01 持久化特性

  文件系统的持久化特性决定了持久化特性在一个操作执行的时候，发生了错误最终文件系统中数据的情况。Paper 中测试了多种的Linux下面的文件系统ext2、ext3、ext4、btrfs、xfs 和 reiserfs。应用层面的Crash一致性很依赖于文件系统的持久化特性。下面举一个🌰，

```
 write(f1, "pp");
 write(f2, "qq");
```

 这个看上去直观的操作在操作的过程中系统发生Crash的情况下最终文件中的结果却不是很直观。这里将写入操作都是为append操作。系统在操作中间发生Crash可能出现的一些情况。如果添加的操作不是原子的，如果文件系统先修改元数据，在修改了文件的元数据之后，就可能出现A、B的情况，A出数据没有实际写入，文件中的数据是垃圾数据，B中则是没有写入完整。另外，现在的文件系统(整个IO栈都是)一般都是乱序的，也就是说上面代码中写入f2在后面，而实际可能是f2先完成操作。保证A情况不出现的性质叫做 size-atomicity，保证B情况不出现的content-atomicity。而乱序，是现在的IO栈默认的情况，谈不上正不正确，只是进行文件操作的时候也特别注意这样的可能的乱序的现象。

![cequal-crash-states](/assets/img/cequal-crash-states.png)

  作者开发了Block Order Breaker (BOB)的工具检测现在的文件系统的持久化特性，下面是一个总结的表，

![cequal-persistence](/assets/img/cequal-persistence.png)

* 原子性，在上面的表中，所有的文件系统所有的配置情况都提供了单个扇区的覆盖写的一致性，一些文件系统的这个性质依赖于底层存储设备写入一个扇区的原子性(现在的存储设备应该都支持这个特性)。在一些新的存储介质上面，比如非易失性内存，它提供byte级别的写入原子性(一般最多为8bytes)，而不是扇区级别。由于添加一个扇区涉及到数据块的写入和文件元数据的修改，所以这里有些文件系统是不能保证原子性的。对于一个or多个块的覆盖写，则比一个or多个块的追加更加难以处理，这个一般地使用日志、CoW等的机制来保障。支持这2个特性的文件系统就少多了。而文件系统的操作一般都能保证一致性(处理ext2)，这个主要得益于日志or CoW技术地使用。
* 顺序，在数据日志的模or sync模式下面，几个文件系统的顺序都是保证，但是结果就是较低的性能。文件系统的延迟分配技术会影响都最佳操作的顺序。



### 0x02 ALICE工具

 ALICE会将文件的操作抽象为逻辑操作，例如write(), pwrite(), writev(), pwritev(),和 mmap()等的操作都会被转化为overwrite 或者是 append这样的逻辑操作。基于Abstract Persistence Model(APM)，利用这些逻辑操作构建依赖关系。

![cequal-apm](/assets/img/cequal-apm.png)

  这些逻辑操作会被拆分为微操作，

```
• write block: A write of size block to a file. Two special arguments to write block are zeroes and random: zeroes indicates the file system initializing a newly allocated block to zero; random indicates an uninitialized block. Writes beyond the end of a file cause data to be stored without changing file size.
• change file size: Changes the size of a file inode.
• create dir entry: Creates a directory entry in a directory, and associates a file inode or directory with it.
• delete dir entry: Deletes a directory entry.
• stdout: Adds messages to the terminal output.
```

基于Abstract Persistence Model(APM)，ALICE将这些文件系统相关的syscall转化为微操作，在此的基础上构建依赖关系图。

```
open(path="/x2VC") = 10
Micro-ops: None 
Ordered after: None

pwrite(fd=10, offset=0, size=1024)
Micro-ops: #1 write   block(inode=8, offset=0, size=512) 
Micro-ops: #2 write   block(inode=8, offset=512, size=512) 
Ordered after: None

fsync(10)
Micro-ops: None 
Ordered after: None

pwrite(fd=10, offset=1024, size=1024)
Micro-ops: #3 write block(inode=8, offset=1024, size=512) 
Micro-ops: #4 write block(inode=8, offset=1536, size=512) 
Ordered after: #1, #2

link(oldpath="/x2VC", newpath="/file")
Micro-ops: #5 create dir entry(dir=2, entry=‘file’, inode=8) 
Ordered after: #1, #2

write(fd=1, data="Writes recorded", size=15)
Micro-ops: #6 stdout(”Writes recorded”)
Ordered after: #1, #2
/*
Listing 2: Annotated Update Protocol Example. Micro-operations generated for each system call are shown along with their dependencies. The inode number of x2VC is 8, and for the root directory is 2. Some details of listed system calls have been omitted.
*/
```

 ALICE工具检查的几个典型的问题，

* Atomicity across System Calls，应用的一个操作必须使用多个syscall来完成，这里的问题就是指这个操作只完成了多个操作的一部分前缀。在这样的情况下应用是否能处理；
* System-Call Atomicity，值得是一个syscall完成是原子性的，ALICE的测试方法是让其前面的sysycall完成之后，在执行这个syscall的之后处于一个中间状态就Crash；
* Ordering Dependency among System Calls，检查不同的syscall之间的顺序依赖关系。
* Durability，在一个文件上面的fsync操作有时候是不够的，在文件夹上面的fsync有时候也是必须的。

另外一个要处理的问题就是dynamic vulnerabilities，一个例子就是在一个循环中执行一个write操作，执行的次数是不一定的。还有就是ALICE并不能保证检查出所有的问题，

```
ALICE is not complete, in that there may be vulnerabilities that are not detected by ALICE. It also requires the user to write application workloads and checkers; we believe workload automation is orthogonal to the goal of ALICE, and various model-checking techniques can be used to augment ALICE.
```

.

### 0x03 应用中易出错的地方

  Paper中测试多个的应用，发现了不少的问题，

![cequal-diagram](/assets/img/cequal-diagram.png)

```
Figure 4: Protocol Diagrams. The diagram shows the modularized update protocol for all applications. For applications with more than one configuration (or versions), only a single configuration is shown (SQLite: Rollback, LevelDB: 1.15). Uninteresting parts of the protocol and a few vulnerabilities (similar to those already shown) are omitted. Repeated operations in a protocol are shown as ‘N ×’ next to the operation, and portions of the protocol executed conditionally are shown as ‘? ×’. Blue-colored text simply highlights such annotations and sync calls. Ordering and durability dependencies are indicated with arrows, and dependencies between modules are indicated by the numbers on the arrows, corresponding to line numbers in modules. Durability dependency arrows end in an stdout micro-op; additionally, the two dependencies marked with * in HSQLDB are also durability dependencies. Dotted arrows correspond to safe rename or safe file flush vulnerabilities discussed in Section 4.4. Operations inside brackets must be persisted together atomically.
```

这里以(A)中问题为例，(i)部分显示的是LevelDB的Compaction操作，(ii)显示的是LevelDB的添加操作。LevelDB这里的一个问题就是添加的log的时候可能导致只是追加了部分数据，一部分是垃圾数据，而LevelDB的恢复垃圾没有处理好这种情况。另外的一个问题在与Compaction的时候，没有正确之持久化目录的目录项，导致文件找不到。



## 参考

1. All File Systems Are Not Created Equal:  On the Complexity of Crafting Crash-Consistent Applications, OSDI'14.