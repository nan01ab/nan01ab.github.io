---
layout: page
title: The ART of Practical Synchronization
tags: [Synchronization]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## The ART of Practical Synchronization 

### 0x00 引言

   这篇文章是关于Adaptive Radix Tree上同步机制的一篇文章，ART是Hyper数据库中主要的索引数据结构。这里介绍了两个同步机制:**Optimistic Lock Coupling** , and**Read-Optimized Write EXclusion (ROWEX)**

```
We synchronize the Adaptive Radix Tree (ART) using two such protocols, Optimistic Lock Coupling and Read-Optimized Write EXclusion (ROWEX). Both perform and scale very well while being much easier to implement than lock-free techniques.
```

  各种的同步机制是一个可拓展性和复杂程度上的一个平衡:

![art-sync-paradigm](/assets/img/art-sync-paradigm.png)

   从这里看到，这里提高的方法中ROWEX和optimistic lock coupling在在没有特殊硬件的支持下这两个方面的平衡是最后的。近些年HTM被一些硬件支持(intel TSX，IBM Power 8, 9)，看起来更加好。 如果这里对ART不了解，可以先看一看[2]；如果对HTM不了解，可以看看网上的介绍文章，也能找到相关的论文；如果对 lock coupling不了解，可以先看一看[3]，(不过这篇论文很老了，比较难找，不过还是可以在网上找到的)，



### 0x01 Optimistic Lock Coupling 

  Lock Coupling同一时刻最多持有2个lock，是一种标准的Btree的同步机制。ART的一个特点有利于使用这样的同步机制，在ART中，一个修改操作最多影响到2个结点，修改的结点(删除，添加 or 更新)和其父结点，这个比Btree的情况更加简单。Optimistic Lock Coupling(OLC)是 Lock Coupling乐观版本，其基本出发点是:

1. 使用read-write locks提高读的性能，对于修改的操作也是先获取read lock，只有在必要的时候在升级为write lock。
2. Optimistic的基本和非Optimistic的版本相似，不过在非乐观的版本中，会造成一些不必要的lock，造成额外的开支。解决的办法和OCC的思想基本相同，先假设这里不会有并发的修改，在真正要修改的时候在做检查，这里使用的是version number的方式。

一个伪代码：

![art-sync-lockup](/assets/img/art-sync-lockup.png)  

  对于insert操作，和获取一般的lock相似，不过要额外做的是增加version number。read操作的时候在必要的时刻在进行检查，如上面的伪代码所示。另外对于delete操作，这里的问题和很多很多并发的数据结构一样，何时回收内存？这里采用的是常见的基于epoch的方法。

 ![art-sync-write](/assets/img/art-sync-write.png)

### 0x02 Read-Optimized Write EXclusion 

​    一般而言，类似OCC方式的机制都存在不能很好的适应write多的情况。 ROWEX是一种介于传统locking和lock free之间的一种同步机制，它能保证read不会被阻塞。当然，它比前面的OLC复杂不少。基本思路如下:

1. 在更新一个节点之前，先回去这个节点的lock，但是这个lock只会阻塞其它的writer。reader从来不获取lock，也从来不做类似version number的方式。这样如何保证reader的正确性呢？这里由writer来解决这个问题；
2. writer的操作必须是原子的，writer也只会在必要的时刻才回去lock。

```
One important invariant of ART is that every insertion/deletion order results in the same tree because there are no rebalancing operations. Each key, therefore, has a deterministic physical location. In B-link trees, in contrast, a key may have been moved to a node on the right (and never to the left due to deliberately omitting underflow handling).
```

在ART中，Radix的优点就是不会像tree结果一样有级联的更新操作。这里先以Node4作为例子:local modifications(本地更新)是比较简单的，只要保证修改节点内部的数据的时候是原子的就可以了。对于可能导致ART结构更改的，Node replacement 的机制是必要的。这个操作分为以下几步:

1. 回去这个节点和其父结点的lock；
2. 创建一个新结点，初始化数据，一部分or全部来自原来的结点；
3. 修改父结点的指针；
4. 标记旧结点为过时的，释放父结点的lock(这个时候不能删除，因为可能有reader正在使用这里面的数据，删除由基于epoch的机制进行)。

Path compression 是另外的一个问题，这个是ART用来节约内存的优化。

```
To change the prefix of the node atomically, we store both the prefix and its length in a single 8 byte value, which can be changed atomically.
```

![art-sync-path-compression](/assets/img/art-sync-path-compression.png)

  这里的一个难点在与现在的硬件只支持一个CAS的操作，没有multi-CAS。为了解决这个问题，在每一个结点中添加一个level字段，保存了包括prefix在内的的结点的高度，结点一旦被创建就不会被修改了，reader根据这些信息适当处理结点被更新之后prefix的问题：

```
 With this additional information, the intermediate state in Figure 4 is safe, because a reader will detect that the prefix at the blue node has to be skipped. Similarly, it is also possible that a reader sees the final state of the blue node without having seen the green node before. In that situation, the reader can detect the missing prefix using level field and retrieve the missing key from the database.
```

.

### 0x03 评估

  这里的具体的信息可以参看[1],

## 参考

1. The ART of Practical Synchronization, DaMoN 2016.
2. The Adaptive Radix Tree: ARTful Indexing for Main-Memory Databases,  ICDE 2013.
3. R. Bayer and M. Schkolnick. Concurrency of operations on B-trees. Acta Informatica, 9, 1977. 