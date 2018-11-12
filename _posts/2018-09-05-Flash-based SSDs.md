---
layout: page
title: Flash-based SSDs
tags: [Storage, New Hardware]
excerpt_separator: <!--more-->
typora-root-url: ../
---



## Flash-based SSDs 



### 0x00 引言

  SSD在个人电脑是逐渐要普及了。在一个很多公司，数据库的机器也是被特殊对待，一般会配备性能杠杠的SSD。这篇主要参考了[1]，这本书是可以免费获取的。



### 0x01 基本结构和操作

   SSD中分为2级的结构，blocks常见的大小是128KB 256KB，另外一个是Page，最常见的大小是4KB。SSD中更加基本的结构是transistor ，一个transistor 可以存储1 2 3 4bits(目前最多一般只有4个)的信息。一个transistor 里面bit越多，性能越差，寿命越短(相对而言，在同样的技术条件下)。



##### 基本操作

 Flash的基本操作有3个:

* Read(page):  读取一个page的数据，这个是SSD比HHD的优点之一，随机读的性能远高于HHD；
* Erase (a block): 很不幸的是，这个是Flash-based SSD的最大最麻烦的一个问题，也极大地影响了SSD和其控制器的设计。在一个page被program之前，SSD必须擦除page所在的整个块。这个是由于SSD存储的原理决定的。
* Program (a page): 将数据写入到一个已经擦除的page中。

.

### 0x02 FTL

  很不幸的是，SSD的寿命也不如HHD，主要的原因是block能被擦出的次数有限:

```
The typical lifetime of a block is currently not well known. Manufac- turers rate MLC-based blocks as having a 10,000 P/E (Program/Erase) cycle lifetime; that is, each block can be erased and programmed 10,000 times before failing. SLC-based chips, because they store only a single bit per transistor, are rated with a longer lifetime, usually 100,000 P/E cycles. However, recent research has shown that lifetimes are much longer than expected.
```

 所以，频繁地写入一个地方会导致SSD很快的损失寿命，这个就是SSD主控要解决的问题之一。SSD修改一个page里面的数据时，并不能直接修改，只能读出原来的数据，修改之后写入新的块。这样的话，物理上的块就不能被直接拿来用，直接使用逻辑上的page。这个也是FTL要实现的功能。

.

### 0x03 A Log-Structured FTL 

​    一个常用的方法就是Log-Structured的 FTL，思路和Log-Structured File System相似(都是这里要解决的问题不同)。基本操作都是写入下一个空闲的page，然后更新mapping table。这里就可以想象，SSD的主控就是一个功能专用的计算机。

  Mapping Table是保存在内存里面的(SSD的内存)，那么这里就遇到了一个和很多存储系统相同的问题：掉电了怎么办？解决办法当然也和很村存储系统一样，使用logging加上checkpoint机制。

 ![flash-ssd-maping-table](/assets/img/flash-ssd-maping-table.png)



#### Mapping Table Size 

  另外一个mapping table需要解决的问题就是这个table的尺寸问题，对于现在的SSD，TB级别的很常见了(咸鱼这台电脑就已经512GB，三星一些1TB级别的性能爆表，价格也比前几年下降了好多)，甚至出现了10TB级别的SSD。对于1TB的mapping table，它的尺寸就是1GB(a single 4-byte entry per 4-KB page results in 1 GB of memory needed the device)。

  有没有想到OS的多级页表，这里也可以使用类似的方法。Page上面有更大的block。现在很多SSD使用了Hybrid Mapping 的机制。在这种方法中，FTL保持几个blocks是已经被擦除的，写的时候直接写这些blocks，这些blocks被称为log blocks。与此同时，FTL必须能以page为单位进行操作，所以它保存了一个page基本的mapping table，但只是对于这些log blocks的，这个table叫做log table。另外一个更大的block级别的table叫做data table。

```
The key to the hybrid mapping strategy is keeping the number of log blocks small. To keep the number of log blocks small, the FTL has to periodically examine log blocks (which have a pointer per page) and switch them into blocks that can be pointed to by only a single block pointer. This switch is accomplished by one of three main techniques, based on the contents of the block
```

.

### 0x04 Garbage Collection 

​    Log-Structured是FTL需要解决的一个问题。对于SSD来说，回收垃圾，然后重新安排存活的page，将空闲的空间放在一块有利于提高性能。这里的基本思路也是和  Log-Structured File System相似，都是读取存活的数据，然后写到另外一个地方，同时将垃圾回收。

   为此，SSD一般预留了一些额外的空间，或者出现了一些240GB的容量之类的。SSD的垃圾回收也是很影响性能的。

```
To reduce GC costs, some SSDs overprovision the device; by adding extra flash capacity, cleaning can be delayed and pushed to the background, perhaps done at a time when the device is less busy. 
```

.

### 0x05 Wear Leveling 

​     前面提到，SSD的block擦除的次数是有限了，一个block坏了就很麻烦。最后的方式就是能将写入分摊到各个block上，这里又要考虑到很多的东西。想详细理解可以参考相关论文。

```
To remedy this problem, the FTL must periodically read all the live data out of such blocks and re-write it elsewhere, thus making the block available for writing again. This process of wear leveling increases the write amplification of the SSD, and thus decreases performance as extra I/O is required to ensure that all blocks wear at roughly the same rate.
```

.

### 0x06 另外一个问题

   这里可以看到SSD内部也是Log-Structured的，对于现在OS的FS来说，大部分都是日志式的文件系统，也适用log来保证cras时的完整性，对于现在的存储系统来说(比如key-value的Level DB, RockDB or常见的数据如MySQL, PostgreQL)，也使用log来保证数据持久化。

  这样一层层的下来，就会做了很多额外的工作。这里就出现了Open Channel SSD，可以在软件层面控制FTL的一些功能，关于这个可以看看[2].



## 参考

1. “Operating Systems: Three Easy Pieces“ (Chapter: Flash-based SSDs) by Remzi Arpaci-Dusseau and Andrea Arpaci-Dusseau. Arpaci-Dusseau Books, 2014. 
2. LightNVM: The Linux Open-Channel SSD Subsystem, FAST 2017