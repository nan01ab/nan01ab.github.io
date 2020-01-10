---
layout: page
title: BTRFS -- The Linux B-Tree Filesystem
tags: [File System]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## BTRFS: The Linux B-Tree Filesystem 

### 0x00 引言

  BtrFS号称是Linux的下一代文件系统，不过感觉现在进展不怎么样，目前性能也渣。虽然有一大堆的新功能和高级特性，不过bug一大堆。从2007年开始开发到现在都11年了，还是这个鬼样子，有种要扶不上墙的感觉。这篇Paper就介绍了BtrFS的基本技术。诸多的新功能:

```
(1) CRCs maintained for all metadata and data;
(2) efficient writeable snapshots, clones as first class citizens; 
(3) multidevice support;
(4) online resize and defragmentation;
(5) compression;
(6) efficient storage for small files;
(7) SSD optimizations and TRIM support.
```

一些BtrFS基本的概念的简单介绍。

### 0x01 基本设计

BtrFS几个基本的知识：

* Page, block，一般是4KB大小连续的区域，在不同的架构上可能存在些许的差别，不过一般都是4KB。Extent是连续的一组Block组成的，可以大大将少元数据的大小，它的大小是不固定了，这个方法改进了类型FFS，ext文件系统中使用bitmap管理Block的一些缺陷，特别是在现在的大容量的磁盘的环境下；
* COW Friendly B-Trees，这个是一些COW的Btree的实现，这个相关的知识在另外一篇论文里面[2]，而且比较复杂，这个具体的算法的实现在这里的暂时略V(^_^)V；

<img src="/assets/img/btrfs-btree.png" alt="btrfs-btree" style="zoom:67%;" />

* Checkpoints，一个Checkpoint可以看作是累积在内存中的更新一次性写入磁盘的一次操作，由一个递增的整数值表明。

* Filesystem B-Tree，BtrFS中所有的元数据都是Btree管理的。Btree的方式解决了类似ext文件系统中存在的一些问题。一个BtrFS文件系统可以看作是一个Btrees组成的森林，包含了下面的一些的Btrees：

  1. Subvolumes。Subvolume是Btrfs支持的一个特性。Subvolumes保存的是用户可见的文件和目录的信息。另外，一个Subvolume由一个Btree表示。Subvolume支持快照可克隆的操作，这个操作会增加一个Btree(利用了上面COW优化Btree的一些特性)；
  2. Extent allocation tree，保存了磁盘空间的分配信息。每一项为一个 extent-items，描述一段磁盘上连续的区域。而且这个区域可能存在多个的应用，也可能只引用了这个区域中的一部分；
  3. Checksum tree，对于每一个extent保存一个checksum item，用于数据正确性的检测。注意这里不是一个extent一个checksum，而是这一个extent里面的一个page一个checksum；
  4.  Chunk and device trees，管理设备。
  5. Reloc tree，在解决碎片问题中使用的结构；

* Fsync，这里主要就是如何优化fsync操作。fsync操作在一些情况下可能造成比较大的性能损耗。BtrFS使用每一个文件一个log tree的方式优化，

  ```
   A naive fsync implementation is to checkpoint the entire filesystem. However, that suffers from high latency. Instead, modified data and metadata related to the particular file are written to a special log-tree. Should the system crash, the log- tree will be read as part of the recovery sequence. This ensures that only minimal and relevant modifications will be part of the fsync code path.
  ```

* Compression，BtrFS直接支持压缩数据保存；

* 快照和克隆，现代的很多的文件系统都支持快照。BtrFS中快照的实现也依赖于前面的COW的Btree的实现和其它的一些机制；

* 软件RAID，BtrFS可以在文件系统层面上实现RAID，支持RAID0，1和10。

### 0x02 多设备支持

  Linux有自己的两个设备管理子系统：device mapper (DM) 和 software RAID subsystem (MD)。但是BtrFS要实现它自己的一些功能，需要自己来管理设备。支持软件的RAID，这个也是Checksum保存在一个单独的Btree上的一个原因。BtrFS将物理的设备划分为chunks，一个经验值就是一个Chunk的大小不要超过设备总容量的10%。BtrFS中，使用一个 chunk tree来保存逻辑的Chunk到物理的Chunk的映射。文件系统其它的部分使用的都是逻辑上的Chunk，extent使用的也是逻辑上的Chunk。这样有一个好处就是可以支持透明的数据迁移。而且一般Chunk的Btree都比较小，可以直接Cache在内存里面。另外BtrFS对于不同的Chunk可以使用不同的分配策略。

### 0x03 去碎片化和迁移

  碎片的问题在各种的文件系统中都会遇到。由于BtrFS是一种COW的文件系统，每次更新都会导致写入到新的位置，这个碎片的问题就变得更加严重。BtrFS去碎片化和迁移中的几点：

* 利用在sync time提交内存中累积的更新的时候优化碎片化的问题。例如延迟分配、合并同一个文件的数据等的方法；

* 使用NOCOW选择，在一些情况下不实用COW，

  ```
  NOCOW makes sense for workloads where COW would be very expensive, for example, database workloads that do random small updates, followed by sequential scans. The normal BTRFS policy would write the blocks to disk in update order, which would cause very bad performance in the sequential scan.
  ```

* 用户可以设置autodefrag选择，BtrFS进行去碎片化的操作。另外BtrFS的Chunk的机制也给这个的实现很大的方便，

### 0x04 评估

  这里的信息可以参看[1]。在SSD上面的性能对比：

<img src="/assets/img/btrfs-perf.png" alt="btrfs-perf" style="zoom:50%;" />

## 参考

1. Rodeh, O., Bacik, J., and Mason, C. 2013. BTRFS: The Linux B-tree filesystem. ACM Trans. Storage 9, 3, Article 9 (August 2013), 32 pages.  DOI:http://dx.doi.org/10.1145/2501620.2501623 .
2. B-trees, Shadowing, and Clones, TOS'08.