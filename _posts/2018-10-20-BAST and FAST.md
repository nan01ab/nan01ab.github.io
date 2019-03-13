---
layout: page
title: BAST, FAST and Superblock-Based FTL
tags: [Storage]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## A Space-efficient Flash Translation Layer for Compactflash Systems, Transactions on Consumer Electronics

 ### 0x00 引言

  这篇是关于FTL Hybrid模式的一篇比较早的Paper。在这篇Paper中，作者提出了一种叫做log block的模式。在这种模式下面，大部分的数据都以Block为粒度来管理，另外的一小部分以Page为粒度来管理。后者称之为Log Blocks，用于临时保存小数据量的写。

### 0x01 基本思路

  在这种模式下，到达一个数据Block中的一个Page的更新操作的时候，一个提前擦除了的日志Block被分配，之后每次这个数据Block的写入操作，就是写入到这个对应的日志Block中。同时，这个逻辑Page和这个物理Page的映射关系会被保存到另外的一个区域。这样的模式下，一个读请求达到的时候，如果请求的Page在日志Block中，就选择返回这个日志Block中的Page，否则返回数据Block中对应的Page。

![bast-merge](/assets/img/bast-merge.png)

  随着日志Block的写入，当这个日志的Blok被写满时，这个之后就要进行Merge操作，即将日志Block的数据写入到数据Block中。这里的操作很直观，首先从空闲的Block中选择一块，将数据Block和日志Block里面的数据选择每个Page最新的写入到新的Block中，而原来的日志Block和数据Block就会归还到空闲Block池子中。之后这些Block会被擦除。这里如果刚好是一个Block里面顺序写入Page，那么这里合并的操作是不需要的，直接用日志Block代替数据Block即可，这样的操作在这里称之为switch操作。这里有些类似于日志式文件系统的一些垃圾回收的操作，不同点在于这里的Block是有限定的等。

![bast-mapping](/assets/img/bast-mapping.png)

  在Merge操作或者是Switch操作之后，都需要去更新一些映射关系。在之前的一些系统中，这些映射关系需要全部保存到SRAM里面。而在这里，Mapping Table保存在指定的一些Blocks中，在需要的时候取出来，这些Blocks称之为Map Blocks。Map Blocks的组织类似于日志Blocks，这些Mapping Tables的映射称之为Map Directory，保存在SRAM中。这样看起来就类似于两级的Page-Table。上图表示了一次Merge操作之后，Mapping Table之前和之后的情况。一个Merge操作实际上会有三个Blocks参与操作，需要更新三个Blocks的映射关系。同样地，Mapping Table使用的数据也要被回收，这里使用了RR的方式来管理，

```
... In our approach, the mapping table is fragmented into a unit of a half of a page so that two mapping table fragments fit into a single page. By limiting the number of log blocks plus free blocks below the maximum number of blocks that a single mapping table fragment can map (128 in our configuration), the updates of three mapping table entries can be performed in a single write operation to the map block. 
```

 Map Directory的构造方式是通过倒着扫描Mapping Blcoks，从最后一个开始倒最旧还使用的一个。FTL是SSD中很重要的数据，这里的更新必须考虑到掉电等情况下的问题，原子性是需要保证的。在这里，元数据的更新被实现为一次的原子写操作，另外失败的写操作可以被保存在另外一个区域的ECC校验值发现。对于很长的顺序写来说，这些数据先保留到一个空闲的Block，然后使用Switch操作来将其转换为数据Blocks。对于一个Block写到一部分被中断的问题，这里就使用了一个bit为来标志其中的Pages是否有效。

### 0x02 评估

 这里的具体信息可以参看[1],

## A Log Buffer-Based Flash Translation Layer Using Fully-Associative Sector Translation
### 0x10 引言

  前面的这篇Paper提出的算法称之为BAST，而这篇Paper提出的是BAST一种优化。基本的策略就是改变BAST中数据Block对应到一个日志Block的策略，FAST可以让一个数据Block关心到所有的日志Block。FAST认为它的策略可以减少不必要的擦除操作、提高日志块的利用率。不过也有另外的一些缺点，比如由于一个日志Block可能保护率多个数据块的数据，这里擦除操作的时候带来的开销可能比较大。

### 0x02 基本思路

  FAST认为BAST的思路存在两个缺点，1. 类似于CPU中Cache的直接映射，这样的方式会导致更加大的“Cache Miss”，这个可能导致Block-Thrashing问题，2. 就是空间不能有效利用的问题，比如一个日志Block被写满的时候，另外一个可能还是很空，但是基于BAST的方法，这个很空的日志Block不能被使用。FAST的方法就是利用了CPU Cache中全相联的思想，一个数据Block对应的日志Block不限制为指定的一个，而是可以是全部的日志Block。基本的思路如下图所示。由于一个数据Block可以关联到多个日志Block，FAST这里引入了一个Sector-Mapping Table，这里的Sector应该就和上面的Page是同一个意思。这个Sector-Mapping Table保存了逻辑Sector和物理Sector之间的映射关系。

![fast-assoc](/assets/img/fast-assoc.png)

   在FAST中，将日志Blocks分为两种类型，一个是用于顺序写的SW Blocks，一个用于随机写的RW Blocks。 在FAST中，一个Sector要被写入的时候，FAST先检查将其写入SW 日志块是否可以看作Switch操作的条件，如果满足，显然就是写入到对应的Sector中。否则写入到RW Block中。这里的Merge的操作基本的处理流程和前面的BAST是一样的，也是拷贝原有的有效数据到新的Block，然后更改映射表，然后将原来的Blocks放回空闲Block池子。这里要分为几种情况，

* 第一种类似于BAST中的Switch操作，SW的日志块里面的Sectors是顺序的，直接用日志Block代替数据Block即可。
* SW日志Block合并。由于FAST在这里做了顺序写的优化。这里可以不请求一个新的Block。比如下图中的a，如果SW Block中，只是Sector 5美元被更新，则将原数据Block的Sector 5拷贝到SW的Block中。接下来就是Switch操作即可。
* 另外一个的处理是FAST合并操作中最麻烦的。它需要将原数据Block和设计到多个RW日志Block的数据拷贝到新的Block。

![fast-merge](/assets/img/fast-merge.png)

### 0x03 评估

  这里的具体信息可以参看[2].

## A Superblock-based Flash Translation Layer for NAND Flash Memory

### 0x20 引言

Superblock FTL可以看作是在BAST 和 FAST的基础上面的一个优化。虽然这篇Paper发表在前面的这篇论文之前，但是这篇Paper参考的关于FAST模式的Paper是一篇05年的。这篇Paper可以看作是一个类似三层结构的的模式。它引入了一个Superblock的概念，将N个数据Block和M个更新Blocks(就是前面的将的日志Block)组成一个Superblock。

### 0x21 基本思路

  Superblock FTL模式是将N + M给物理Blocks映射为一个逻辑上为N个Blocks。虽然Superblock-based模式看起来像是三层是的结构，这里实际上还是一个两层式的结构。如下图所示，Logical Block Number在这里被分为了两个部分，一个是Superblock Number，另外一个是PGD Number。这里实际上就是将原来的Block Number分为了两个部分，一部分用于Superblock寻址，一部分用于Superblock内部的寻址。Superblock模式像是BAST和FAST模式的一个这种，BASR是一个直接映射的关系，FAST是一个全映射的关系，而Superblock模式像是一个将Fully Association限制在一个Superblock的FAST。

![superblock-addr](/assets/img/superblock-addr.png)

  Paper中观察到了时间上的局部性和空间上的局部性。时间上的局部性是指同一个Block里面的Page如果一个被更新，其它的更加可能在不久的将来被更新。空间上的局部性是指相邻逻辑Block里面的Page更加可能在一个更新之后另外的也被更新。Superblock模式可以能够更好地利用这些局部性。这里在一个Superblock里面虽然使用了全相联的方式，但是这里有一些和FAST模式不同的地方。不同于FAST中SW和RW的设计，Superblock模式会将写请求写入到一个更新Block中，而不去关心它属于哪一个逻辑的Block。

### 0x22 评估

  这里的详细信息可以参看[3].

## 参考

1. A Space-efficient Flash Translation Layer for Compactflash Systems, Transactions on Consumer Electronics 2002.
2. A Log Buffer-Based Flash Translation Layer Using Fully-Associative Sector Translation, TECS '07.
3. A Superblock-based Flash Translation Layer for NAND Flash Memory, EMSOFT '06.