---
layout: page
title: IndexFS and LocoFS
tags: [Distributed, Storage]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## IndexFS: Scaling File System Metadata Performance with Stateless Caching and Bulk Insertion

### 0x00 引言

 这Paper是关于利用KVS作为分布式文件系统的元数据存储的。之前就有一些Paper设计了利用类似LevelDB的Key-Value来实现单机的文件系统。比如TableFS[2]是利用LevelDB实现的一个用户空间文件系统，TableFS的论文里面只是简单的介绍了一些FS层次结构到KV的映射，但是没有谈到解决重命名以及文件夹移动等的问题，这个问题比较难处理。后面的BetrFS则将类似的思路般到了内核空间，相关的几篇Papers也重点讨论了重命名、移动文件文件夹等操作的实现。这里的IndexFS则是将环境放到了分布式文件系统下面，主要是用KVS存储分布式文件系统的元数据。

![indexfs-arch](/assets/images/indexfs-arch.png)

### 0x01 基本架构

  IndexFS的基本架构如上图，整体上还是类似于GFS的架构。主要的变化就是元数据保存到了Key Value存储系统中，

* 对于元数据保存到KV存储系统中，Key为Parent directory ID, Hash(Name)，Value为Name, Attributes, Mapping\|File Data\|File Link。Value的后面三种可以的情况根据这个文件是文件夹、小文件还是大文件来确定。另外常见的一个优化一样，如果是小文件，IndexFS就会直接将这些数据内联到元数据中。对于大的目录，IndexFS会使用动态分区的方式，分区的方法和GIGA+的一样(可以参看其论文)，基本思路就是先将目录项映射到一个大的hash空间内，然后根据每个分区内的负载的情况在对其进行合适的分区。

* 处理基本的Key-Value映射之外，IndexFS将一部分更加常用的原属分离出来，保存到另外一个小的Table中，其它的数据保存到一个大的Table中。IndexFS将其称之为Column-Style Table。这个也是一个优化，主要的考虑就是文件的元数据并不少，但是里面常用的并不多，而且不同的文件操作常常用的是不同的元数据。

  ![indexfs-column-style](/assets/images/indexfs-column-style.png)

* IndexFS在创建文件上面的一个优化是一次“攒”其多个文件创建的操作，然后一次性批量处理。在分区分裂时候的优化利用了SSTable的一些特点[1]。

* 另外，IndexFS针对这样的环境下面设计了对应的缓存策略。

### 0x02 评估

  这里的具体的信息可以参看[1],

## LocoFS: A Loosely-Coupled Metadata Service for Distributed File Systems

### 0x10 引言

  LocoFS是在IndexFS上面的一个优化操作。LocoFS发现IndexFS的设计能够获得的吞吐和原始的Key-Value操作相差很大。它的主要思路有，1. 目录的元数据和文件的元数据分开保存，2. 文件的不同的元数据访问模式存在很大的区别，这里LocoFS将文件元数据根据访问模式分开保存，3. 另外，前面的IndexFS没有提到rename之类的操作怎么优化，而LocoFS利用赋予一个UUID来优化这个问题，

```
Evaluations show that LocoFS with eight nodes boosts the metadata throughput by 5 times, which approaches 93% throughput of a single-node key-value store, compared to 18% in the state-of-the-art IndexFS.
```

### 0x11 基本思路

 LocoFS可以看作是从前面的IndexFS改进而来的一个架构。架构上一个主要的变化就是目录的元数据和文件的元数据分开保存。在文件系统层次式的结构中，目录的操作是比较耗时的，而前在IndexFS的设计中，访问一个目录下面的一个文件可以需要访问多个元数据服务器。在LocoFS中，它认为目录的元数据是远少于文件的元数据的，一次将目录的元数据只保存到一个元数据服务器上面，就可以直接在一个服务器上面就完成查找一个路径下面文件的操作。另外和IndexFS一样，文件的元数据是保存到多个的元数据服务器上面的，

![locofs-arch](/assets/images/locofs-arch.png)

LocoFS的设计还有另外的一些特点，

* 因为文件和目录元数据分开保存的逻辑，LocoFS将inode分为f-inode和d-inode。分别保存到对应的元数据服务器中。

* 在LocoFS中，每一个目录会被赋予一个uuid，下面的文件使用directory_uuid + file_name作为可以保存到KVS里面。

* 扁平化的目录树设计。在一般的文件系统设计中，目录会保存每一个目录项的信息，通过dirent-inode链接来形成文件系统的层次结构。而在LocoFS中改变了这样的设计，LocoFS这样设计是因为这样的带有强依赖性的设计会来不小的overhead，

  ```
  ... Rather than stored with the directory metadata, the dirents are respectively stored with the inodes of files or subdirectories. Specifically, each subdirectory or file in a directory stores its dirent with its inode. As shown in the bottom half of Figure 4, the main body of a directory or file metadata object is its inode. Each inode keeps the corresponding dirent alongside. As a consequence, these metadata objects are independently stored in a flat space.
  ```

  ![locofs-flatted-tree](/assets/images/locofs-flatted-tree.png)

* 文件系统中一个很重要的部分就是权限的操作，比如creat，remove和open等的操作。IndexFS中的设计带来的问题就是目录的操作可能操作需要访问多台服务器，LocoFS这样的设计的缺点就是每次都要访问目录的元数据可能给元数据服务器带来很大的眼里。LocoFS这里使用了一些缓存的策略来缓解这个问题。

  ```
  ... When the client creates a file in the same directory, accesses to its directory metadata can be met locally. For the path traversal operations, LocoFS caches all the directory inodes along the directory path. This also offloads the DMS’s traffic.
  ```

  (这里有个可以有更好的策略，可以从IndexFS的设计和LocoFS的设计综合考虑，另外假设不同层次文件目录的元书访问的频率存在较大的差别入手).

* 这里和IndexFS的出发点类似，也是文件的元数据保存的一个优化。LocoFS将文件的元数据细粒度的分成不同的部分，分开保存。一个文件的操作只需要访问需要的部分，减少了操作的开销，

  ```
  ... We can see that most operations (e.g., create, chmod, read, write and truncate) access only one part, except for few operations like getattr, remove, rename. For instance, the chmod operation only reads the mode, uid and gid fields to check the access control list (ACL), and then updates the mode and ctime fields, accessing only the access part...
  ```

* LocoFS另外一个优化就是前面的IndexFS没有怎么提到的rename的问题。LocoFS认为rename操作在文件系统中的操作只占一个非常小的部分。前面提到的LocoFS的在KVS保存文件元数据的Key的策略也是其的一个优化之一。rename操作会改变文件/文件夹的名字，但是不会改变其uuid，这样就可以避免rename操作导致的大量的数据改动操作。

### 0x12 评估

  这里的具体信息可以参看[3],

![locofs-perf](/assets/images/locofs-perf.png)

## 参考

1. IndexFS: Scaling File System Metadata Performance with Stateless Caching and Bulk Insertion, SC '14.
2. TABLEFS: Enhancing Metadata Efficiency in the Local File System, ATC '13.
3. LocoFS: A Loosely-Coupled Metadata Service for Distributed File Systems, SC '17.