---
layout: page
title: NOVA-Fortis -- A Fault-Tolerant NVMM File System
tags: [New Hardware, File System]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## NOVA-Fortis: A Fault-Tolerant Non-Volatile Main Memory File System

### 0x00 引言

  这也是SOSP'17上面一篇关于NVM上面文件系统设计的Paper。NOVA-Fortis是在之前的NOVA文件系统之上功能增强发展而来，

```
NOVA-Fortis’ reliability features consume 14.8% of the storage for redundancy and reduce application-level performance by between 2% and 38% compared to the same file system with the features removed. NOVA-Fortis outperforms DAX-aware file systems without reliability features by 1.5× on average. It outperforms reliable, block-based file systems running on NVMM by 3× on average.
```

 另外这篇Paper可以说是做的很精美了，2333。

### 0x01 快照

  快照是NOVA-Fortis文件系统在NOVA文件系统上面的一个增加的功能。NOVA-Fortis可以支持系统在运行的时候创建快照，可以挂载一个快照作为一个只读文件系统，也可以回滚到过去的一个快照。NOVA-Fortis支持无线数量的快照，另外，NOVA-Fortis面向的是NVMM存储介质，它也实现了应用在使用DAX mmap()的时候也可以产生一个一致性的快照。

#### 建立一个快照和恢复到一个快照

  NOVA-Fortis快照的实现主要是维持了系统的一个全局的快照ID，另外在每一个Log项里面保存了一个本地的快照ID。建立一个快照的方式就是增加这个全局的快照的ID，将之前的有效的快照ID保存到一个list中。建立一个快照在NOVA-Fortis中是一个常数时间的操作，只要目前没有文件正在使用mmap。使用过去文件系统的一个快照时，NOVA-Fortis挂载文件系统为一个只读的文件系统。读取一个打开读取文件时，只会处理那些本地的快照ID小于等于使用的快照ID的数据。关于NOVA-Fortis快照设计的另外几点，

* 快照管理，NOVA-Fortis使用Snapshot Manifest来保存一些关于快照的一些信息。这个Manifest都会有指向log项的指针，另外还包含了[create snapshot ID, delete snapshot ID)这样一个结构代表来这个快照存在的时间。下面的例子说明了NVOA-Fortis采用的思路：初始的时候快照ID为0，写入了2块8KB的数据。之后建了一个快照，覆写了0-4K的数据，这个log被本地的快照ID就为1。之前的数据由于创立的快照ID和死亡的快照ID不相同，所以他们之前的数据不能删除。(c)中有建了一个快照。并覆写了之前两块的数据的内容，使用前面的数据块也要添加上Snapshots Manifest。删除一个快照的时候，只有这个数据所有的可见的快照都删除了之后才能去删除数据，比如图d中的最开始的0-4K的数据块。为了提高性能，这些Snapshots Manifest信息是保存到内存里面的，在关机的时候这些数据会被写入到NVMM中，但是如果是掉电之类引起的关机，就需要从文件系统中重新构建这些信息(或许也可以采用lazy的方法)；

<img src="/assets/img/novaf-snapshots.png" alt="novaf-snapshots" style="zoom: 50%;" />

* 快照与DAX mmap()’d Files，这里要处理的问题就是memory-mapped file直接使用load store的一段操作的完整性。这里处理的一个简单的思路就是COW，不过仅仅是这样也会存在问题。主要的原因就是标记数据只读，标记相关元数据(一些标记位)只读，和一个标记操作的顺序的问题，

  ```
  If NOVA-Fortis marks D’s page as read-only before the program updates D, and marks V ’s page as read-only after the program updates V , the snapshot has V equal to True, but D with its old, incorrect value.
  ```

  NOVA-Fortis使用的方法就是阻塞一个读取read-only的操作直到所有的相关的Pages都被标记完成位read-only，这样久避免了更新操作插入到这些标记这些不同的Pages的操作中去。

### 0x02 处理数据错误

 NOVA-Fortis的这篇Paper中关于如何处理这些数据错误的问题有大量的篇幅[1]。这里就只简要总结一下，  

* 检测和纠正介质的错误，这里的方法采用了检测内存错误一样的方法；

* 使用Tick-Tock来保护元数据，这里实际上就是使用副本的方式，在一些文件系统中也使用了相关的方式来保存文件系统元数据，比如BtrFS中就使用软件RAID来保存元数据。元数据保存了2份。只有在主副本持久化了才去写次副本。同时使用memcpy_mcsafe()在拷贝数据时检测数据错误。

  ```
  To reliably access a metadata structure NOVA-Fortis copies the primary and replica into DRAM buffers using memcpy_mcsafe() to detect media errors. If it finds none, it verifies the checksums for both copies. If it detects that one copy is corrupt due to a media error or checksum mismatch, it restores it by copying the other.
  ```

<img src="/assets/img/novaf-layout.png" alt="novaf-layout" style="zoom:67%;" />

* NOVA-Fortis采用了RAID-4的方式来保存文件的数据。另外对于DAX-mmap’d 的数据使用Caveat DAXor的保护方式。

* Relaxing Data and Metadata Protection，为了提高性能，也可以在一些情况下使用一些放松保护的方式，

  ```
  In relaxed mode, write operations modify existing data directly rather than using copy-on-write, and metadata operations modify the most recent log entry for an inode directly rather than appending a new entry. Relaxed mode guarantees metadata atomicity by journaling the modified pieces of metadata.
  ```

* NOVA-Fortis对于内存中大部分的数据结构也使用了checksum的方式来保证正确性；

* 另外还有处理Scribbles[1]……....

### 0x03 评估

  这里的具体信息可以参看[1]，这部分这篇Paper做得非常精美了鸭🦆。

<img src="/assets/img/novaf-perf.png" alt="novaf-perf" style="zoom:50%;" />

## 参考

1. NOVA-Fortis: A Fault-Tolerant Non-Volatile Main Memory File System.SOSP ’17.