---
layout: page
title: Many Tries
tags: [Data Structure]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## Many Tries

### 0x00 引言

  前面的SIGMOD‘18 上面的文章提出来一个新的数据结构HOT，是一种trie的变体。里面提到了不少相关的结构，总结如下。另外这些结构能够应用的范围都比较窄。

### 0x01 Patricia trie 

  Patricia trie是出现的非常早的一种trie的变体。Patricia trie的论文也很长。这里主要关注它的一个优化就是将只有一个字节点的数据直接保存到父节点中。就是对于一条长链时的节点，

```
-> A -> B -> C -> D
每一个节点都只有一个子节点，那么久可以直接保存为一个节点，
-> ABCD.
```

这种优化方式在一些情况下能取得很好的优化的效果。在adaptive radix tree中也使用了这样的方法优化。

### 0x02 BURST-Trie 

  BURST-Trie是一种Trie和其它数据结构结合而来的。BURST-Trie的内部节点就是一般的trie的节点，而外部节点时其它的数据结构，这个结构并不固定是哪一种。 BURST-Trie这么设计的原因也是利用叶子节点来减少trie内存的使用。这里的叶子节点可以就是一棵平衡树或者是类似的节点即可。在这个叶子节点到达一定大小的时候，会被“burst“，即这个叶子节点拆分为更多的叶子节点，同时添加必要的中间的trie节点。 BURST-Trie在一些倒排索引中有使用。

### 0x03 HAT-Trie 

  HAT-Trie 也可以就看作是BURST-Trie 的一个变体。它使用的叶子节点的结构时hash table。在HAT-Trie 中有两种类型的叶子的hash table，

* pure，在这个hash table中的数据共享一个前缀，保存的时候可以去除这个前缀；
* hybrid，hybrid的hash table包含来来自多个内部节点的数据，为了区分来自哪一个内部节点，需要一些前缀信息。使用hybrid的好处就是对这类节点burst操作的时候直接拆分即可，另外就是可以进一步减少trie占用的内存。

  HAT-Trie的一个缺点也是来自使用hash table，由于hash table是无序的，在处理点查询的时候还好，处理范围查询的时候可以处理起来比较麻烦。

### 0x04 KISS-Tree

  KISS-Tree是一种十分简单，易于实现，Larch-free的实现起来也比较简单(还是因为它足够简单)。KISS-Tree的主要缺点就是应用范围很有限，只能应用在32bit的整数的情况下，KISS-Tree在它适用的范围内还是非常优秀的，也可以在它思路之上拓展一下适用的范围。它将32bit的整数分为3个部分，有点像现在的多级页表的实现，但是它每一层的长度和实现的方式不同。

<img src="/assets/img/kisstree-overview.png" alt="kisstree-overview" style="zoom:67%;" />

* 第一次层是虚拟的，长度为16bit，用来直接定位第二层；
* 第二层长度为10bit，中间每一个元素为4byte，这样刚好时一页。这里利用了现在的操作系统按需分配的机制来避免分配没有使用的内存。另外一个就是指针压缩，需要把8byte的指针压缩为4byte，KISS-Tree这里使用方法是：32bit的值中间的26bit用来定位第三层中的哪一块，而剩下的6bit保存这个块的长度，这样直接就在32bit里面保存了2个部分的数据；
* 第三层最多64个值，直接使用一个64bit的整数标识；

### 0x05 x-fast trie

   x-fast trie是bitwise trie的一种变体[4]。 x-fast trie的值都保存在叶子节点上面，如果一个内部节点没有左子节点，那么它会保存一个指向右子树中最小的子节点。如果一个内部节点没有右子节点，那么它会保存一个指向左子树中最大节点的指针。另外，每一个叶子节点会保存一个指向前驱和后继的指针，形成一个双向链表，方便范围查找。另外，每一层的节点会保存为一个hash table。

<img src="/assets/img/Xfast-trie-exampl.png" alt="Xfast-trie-exampl" style="zoom:50%;" />

​                [图片来自维基百科]

### 0x06 y-fast trie

   x-fast trie这样的多个的指针加上hash table会消耗大量的内存。y-fast trie是其在内存使用上面的一个改进。和前面的BURST-trie的思路一样，它也是在上面是一个x-fast trie，下面的部分使用查找树。

<img src="/assets/img/Y-fast-trie.png" alt="Y-fast-trie" style="zoom:50%;" />

​                       [图片来自维基百科]

### 0x07 z-fast trie

  emmmmm，其实没见过这个东西。

## 参考

1. Burst Tries: A Fast, Efficient Data Structure for String Keys.
2. HAT-trie: A Cache-conscious Trie-based Data Structure for Strings.
3. KISS-Tree: Smart Latch-Free In-Memory Indexing on Modern Architectures, DaMoN 2012.
4. https://en.wikipedia.org/wiki/X-fast_trie，维基百科.
5. https://en.wikipedia.org/wiki/Y-fast_trie，维基百科.
6. WORT: Write Optimal Radix Tree for Persistent Memory Storage Systems, FAST'17.

