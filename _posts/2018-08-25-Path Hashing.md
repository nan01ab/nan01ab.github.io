---
layout: page
title: A Write-Friendly and Cache-Optimized Hashing Scheme for NVM Systems
tags: [Data Structure, New Hardware, Key-Value]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## A Write-Friendly and Cache-Optimized Hashing Scheme for Non-Volatile Memory Systems 

### 0x00 引言

  非易失性内存(NVM)是现在的一个研究热点。对于现在存在的一些hash table的算法都不能很好的适应NVM的环境，其主要原因就是NVM写的成本相对来说比较高，而常规的设计会造成很多额外的写操作。这篇paper的主要目的就是减少额外的写操作，同时为cache优化。



### 0x01 基本思路

  这篇paper使用的方法叫做Path hashing ，方法理解起来很简单，文章中的一段话加上一幅图即可：

```
  Storage cells in the path hashing are logically organized as an inverted complete binary tree. The last level of the inverted binary tree, i.e., all leaf nodes, is addressable by the hash functions. All nodes in the remaining levels are non-addressable and considered as the shared standby positions of the leaf nodes to deal with hash collisions. When hash collisions occur in a leaf node, the empty standby positions of the leaf node are used to store the conflicting items. Thus insertion and deletion requests in path hashing only need to probe the leaf node and its standby positions for finding an empty position or the target item, resulting in no extra writes.
```

 基本思路就是出路原来位置槽的部分外，还有额外的共享的standby cells，这些cells使用了解决hash冲突的，同时又不造成额外的写操作。

![path-hashing-arch](/assets/img/path-hashing-arch.png)

#### Position Sharing 

  倒置的树形结构，除了最高的一层外，更高的层的standby cell是有多个slot共享的，越低共享的slot就越多。可想而知，越向下数量越少就节约了空间。如果都使用一样的节点数量，达到下面层的数据项是越来越少的，这样就会浪费空间。

#### Double-Path Hashing 

  一盒hash函数的情况下，每一个key可以放置的位置是L+1个(L是高度)，一个基本的思路使用2个hash函数，这样的话可选择的位置就增加了接近2倍。但是实际上这样的方法并不是很好，两个hash有可能有不少的重叠的地方。更加简单更加好的方法就是一个hash函数计算在前半部分的slot，一个计算在后半部分的slot。

#### Path Shortening 

  去除没有数据的level，消除了不必要的查询。



![path-hasing-phy](/assets/img/path-hasing-phy.png)

### 0x02 算法

   Path hashing的基本算法和普通的hash table没有区别，算法也简单直观，直接就直接给出paper中的描述就可以了: 插入，可以插入的位置都要考虑到。我们这里也发现了path hasing的一个缺点，没有rehash的逻辑。(个人认为这里有一个比较好的解决方法，不知道行不行得通).

![pash-hasing-insert](/assets/img/pash-hasing-insert.png)

  其它的query，delete操作都比较简单，就不重复了。

### 0x03 缓存优化

  简而言之，就是将一些"level""压"到一起：

![path-hashing-cache](/assets/img/path-hashing-cache.png)

简单有好玩的新思路，good idea。

## 参考

1. A Write-Friendly and Cache-Optimized Hashing Scheme for Non-Volatile Memory Systems，TPDS 2018.