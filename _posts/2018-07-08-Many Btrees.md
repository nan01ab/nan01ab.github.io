---
layout: page
title: Many B-trees
tags: [Data Structure]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## Many B-trees

### 0x00 引言

 Btree及其变体可以是是最重要和使用最广泛的较复杂的数据结构之一了。对Btree的改进页非常多，从传统的面向机械硬盘等存储介质的优化，到为内存实现的优化，再到现在的为非易失性内存的优化以及写优化等等的很多很多的变体。这里就简单地总结了若干的Btree的变体。在之前的看过的论文中包括了一些B-Tree的变体: Masstree、PLAM、Bz-tree以及Bw-tree。



### 0x01 Bε-tree

 Bε-tree是Btree的一个写入优化的版本(保存在磁盘上时)，它的思路的是在一个内部节点不仅仅保存Key和指向下一层节点的指针。还包含了一部分的Buffer空间。数据添加的时候先添加到这个Buffer，在Buffer中数据的数量达到一定量的时候将其“下推”。Bε-tree这样的做的优化就是使得小数据量的写入可以先集中写，也可以实现在一个地方将写入累积其它，达到一定量的时候在写入下层对应的位置。 Bε-tree这样的做的缺点就是结构的复杂程度会提高，实现起来更加复杂。另外就是范围操作的时候可能麻烦不少。Bε-tree在BetrFS中得到了应用[3]，

![betree-overview](/assets/img/betree-overview.png)

### 0x02 Write-Optimized B-Tree

  Write-Optimized B-Tree出自论文[4]，它是在B+tree的一些优化。它在B+tree的修改体现在下面的几点：

* 在B+tree中，当叶子节点迁移到一个新的位置之后，由与他的兄弟节点保存了它的Page ID的信息。这个信息用于加快范围查找。缺点就是可能导致级联的更新操作，移动一个节点的时候这样只需要更新父节点的对应的信息即可。WO-Btree则去除了这个兄弟节点的指针，引入了fence key的概念。节点保护了两个fence key，标识了这个节点内Key的范围；
* 在去除了兄弟指针之后，WO-Btree就不同担心级联更新的问题，页面移动变得更加方便和低成本了。WO-Btree写入更新之后的Page是写入到一个新的位置，而不是就地更新，优点类似于COW。
* 去除了兄弟节点可以对范围查找操作又一些不利。

![wo-bree](/assets/img/wo-bree.png)

### 0x03 NV-Tree

  NV-Tree为Btreee在NVMM环境下的一个优化的变体。主要的思路就是3点：

* 选择性的强制数据一致性。在直接寻址NVMM的时候，要经常使用flush缓存以确保数据都持久化到NVMM上面。这这个操作对性能的影响比较大。NV-tree的做法是只对叶子节点确保这样的数据的强制一致性，而对内部节点，则认为内部节点完全可以从叶子节点重新构建处理，这里就发送了它的数据一致性；
* 数据项在叶子节点无序保存，这个的设计简化了叶子节点一些操作，同时也降低了一致性的成本。
* 内部节点使用缓存优化的格式，在NV-tree中，所有的内部节点按缓存行对齐保存在连续的内存中，使用偏移寻址而不是直接使用指针。这样的一个缺点就是内部节点需要添加or删除的时候会重建整个的内部节点。

![nvtree-arch](/assets/img/nvtree-arch.png)

### 0x04 FP-Tree

  FP-Tree在不少地方的设计和NV-Tree类似，另外FP-Tree还加入了其它的一些优化。 FP-Tree的做法是将内部节点保存在传统的内存中，而叶子节点保存在非易失性内存中。FP-Tree在一般的B+Tree的基本上的优化的几个点：

*  Selective Persistence， FP-Tree的内部节点保存在DRAM中，而叶子节点保存在非易失性内存中；
* Fingerprinting，Fingerprinting是叶子节点Key的一个1byte的hash值，连续地保存在叶子节点的前面。在FP-Tree中，和NV-Tree一样，它的叶子节点也是没有排序的，使用这个方法可以加快查找的速度。这种方式在一些Cuckoo Hash Table的设计中也有类似的思路。
* Selective Concurrency，对于叶子节点，FP-Tree使用细粒度的锁，而对于内部节点，使用HTM。

```
查找算法：
while TRUE do 
   speculative_lock.acquire(); 
   Leaf = FindLeaf(K);
   if Leaf.lock == 1 then
     speculative_lock.abort();
     continue;
   for each slot in Leaf do
       set currentKey to key pointed to by Leaf.KV[slot].PKey
       if Leaf.Bitmap[slot] == 1 and Leaf.Fingerprints[slot] == hash(K) and currentKey == K then
          Val = Leaf.KV[slot].Val;
          Break; 
   speculative_lock.release(); 
   return Val;
```

 另外，在添加操作的时候，为了保证咋节点分裂的时候保证一致性，FP-Tree使用了micro-log的方式(删除的时候也会使用)，

```
Algorithm 3 SplitLeaf(LeafNode Leaf):
1: get μLog from SplitLogQueue;
2: set μLog.PCurrentLeaf to persistent address of Leaf; 3: Persist(μLog.PCurrentLeaf);
4: Allocate(μLog.PNewLeaf, sizeof(LeafNode))
5: set NewLeaf to leaf pointed to by μLog.PNewLeaf; 6: Copy the content of Leaf into NewLeaf;
7: Persist(NewLeaf);
8: (splitKey, bmp) = FindSplitKey(Leaf);
9: NewLeaf.Bitmap = bmp;
10: Persist(NewLeaf.Bitmap);
11: Leaf.Bitmap = inverse(NewLeaf.Bitmap);
12: Persist(Leaf.Bitmap);
13: set Leaf.Next to persistent address of NewLeaf; 14: Persist(Leaf.Next);
15: reset μLog;
```

.

### 0x05 Write-Atomic B+-Trees (a.k.a. wB+-Trees)

  wB+-Tree则是将内部节点和叶子节点都保存到非易失性内存之上。它排序的做法不是直接对数据项本身进行排序，而是使用了一个slot array，里面保存的是实际数据的index。排序的时候排序这个slot array。因为这个index array的大小一般都元小于实际的数据项。另外的一个和前面的FP-Tree一样，wB+-Trees也使用bitmap。

```
1: procedure INSERT2SLOTBMP ATOMIC(leaf, newEntry)
2: if (leaf.bitmap & 1 == 0) /* Slot array is invalid? */ then
3:   Recover by using the bitmap to find the valid entries, building the slot array, and setting the slot valid bit;
4: end if
5: pos= leaf.GetInsertPosWithBinarySearch(newEntry);
6: /* Disable the slot array */
7: leaf.bitmap = leaf.bitmap - 1;
8: clflush(&leaf.bitmap); mfence();
9: /* Write and flush newEntry */
10: u= leaf.GetUnusedEntryWithBitmap();
11: leaf.entry[u]= newEntry;
12: clflush(&leaf.entry[u]);
13: /* Modify and flush the slot array */
14: for (j=leaf.slot[0]; j≥pos; j - -) do
15: leaf.slot[j+1]= leaf.slot[j];
16: end for
17: leaf.slot[pos]=u;
18: for (j=pos-1; j≥1; j - -) do
19: leaf.slot[j]= leaf.slot[j];
20: end for
21: leaf.slot[0]=leaf.slot[0]+1;
22: for (j=0; j≤leaf.slot[0]; j += 8) do
23: clflush(&leaf.slot[j]);
24: end for
25: mfence(); /* Ensure new entry and slot array are stable */
26: /* Enable slot array, new entry and flush bitmap */
27: leaf.bitmap = leaf.bitmap + 1 + (1<<u);
28: clflush(&leaf.bitmap); mfence();
29: end procedure
## Insertion to a slot+bitmap node with atomic writes
/*
During failure recovery, a slot+bitmap node may be in one of three consistent states: (i) the original state before insertion, (ii) the original state with invalid slot array, or (iii) successful insertion with valid bitmap and valid slot array.
*/
```

.

### 0x06 clfB-tree: Cacheline Friendly Persistent B-tree for NVRAM

  clfB-tree也是一种为非易失性内存设计的B-tree的变体。它的设计体现在2点：

* Differential Encoding， clfB-tree为了让一个节点的大小能够控制在一个缓存行的大小又尽可能地在一个结点内多放入数据， clfB-tree使用一个差分编码的方法。如下面的图所示，最左边的结点保存完整的信息，然后根据与最左边结点的数据的差值来计算后面的Key和指针要占用的空间，这种方式在一些时间序列数据中是一种常用的方法。在同一个节点中的数据一般都是比较接近的，这个也是一种比较有效的方法；

* Failure-Atomic Cache Line Write，这一点是利用Restricted Transactional Memory实现原子(失败)写入。能够利用write combining store buffer，并减少flush缓存的次数，

  ```
  procedure
  atomicCacheLineWrite(src,dest)
   while 1 do
    int status = _xbegin();
    if status == _XBEGIN_STARTED then
      memcpy(dest, src, 64); 
      _xend();
      break;
    else
       ;
    end if
  endwhile 
  clflush(dest)
  ```

![clfbtree-node-2](/assets/img/clfbtree-node-2.png)

## 参考

1. NV-Tree: Reducing Consistency Cost for NVM-based Single Level Systems, FAST'15.
2. FPTree: A Hybrid SCM-DRAM Persistent and Concurrent B-Tree for Storage Class Memory, SIGMOD'16.
3. BetrFS: Write-Optimization in a Kernel File System, TOS'15.
4. Write-Optimized B-Trees, VLDB'04.
5. clfB-tree: Cacheline Friendly Persistent B-tree for NVRAM, TOS'18.
6. Persistent B+-Trees in Non-Volatile Main Memory, VLDB'15.