---
layout: page
title: Facebook Gorilla
tags: [Storage]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## Facebook Gorilla

  Gorilla是Facebook公司内部的时序数据库产品。Facebook在世纪的使用过程中发现现有的产品不能满足对Facebook超大数据量的处理要求，开发了Gorilla这一样一个产品，通过应用多种压缩方法、将数据放在内存之中，Gorilla获得了73x的延时提升、14x的吞吐量提升。

### 介绍&背景

   2013年Facebook就开始使用一套基于HBase的时序数据库，但是随着Facebook的发展，这套系统以及不能满足未来的负载，90%的查询已经长达几秒。一个对几千个时间序列的查询要消耗几十秒的时间来执行，而在稀疏的更大数据集上查询通常会超时。HBase被设置为写入优化，现在在其中已经保存了2PB的数据，查询的效率不高，又不太好完全更换系统。所以，Gorilla讲注意力转移到给现有的系统在一个in-memory的cache。这里数据的特点就是新数据一般是热点数据，所以选择奖近段时间内的数据cache，就能很好地提高性能。
   Memcache是Facebook大规模使用的一个缓存系统，但是将memcache应用在这里，追加新数据到已经存在的时间序列中要消耗一个read／write周期，给memcache造成很大的压力，所以需要一种更好的解决方案。注：这里没怎么看懂，原文是：

```
  We also considered a separate Memcache [20] based write-through cache but rejected it as appending new data to an existing time series would require a read/write cycle, causing extremely high traffic to the memcache server. We needed a more efficient solution. 
```

对于Gorilla，主要有以下的要求：

```
    1. 2billion的unique的时间序列id，以string表示；
    2. 每秒700million的data points（timestamp加上value）写入；
    3. 保存数据26小时；
    4. 超过40k的QPS；
    5. 读成功在1ms以下；
    6. 支持15s的时间粒度；
    7. 两个用于灾难恢复的不一致的内存副本；
    8. 单服务器故障时也能保证读取；
    9. 可以快速扫描所有的内存数据；
    10. 可以支持每年至少增长2x；
```

现有的OpenTSDB、InfluxDB、Whisper都不能满足这些要求；



### 架构

#### 数据保存方式

  Gorilla中数据的基本单元时一个Timestamp+value的组合，其中，Timstamp时一个654bit的整数，value是一个double类型的浮点数，一个这样基本单元的stream称为data stream。这个的一连串的数据被压缩保存，称为一个block，block存为一个header加上后面被压缩的数据。

#### 时间戳压缩

  根据Gorilla论文的描述，16bytes的数据被压缩到1.37bytes，压缩了约12倍。其中，对于64bit的时间戳使用的压缩方法是delta-of-delta的方式，基本的原理如下：Gorilla不会保存整个世界戳，因为时间戳大部分的bit位都是相同。举个例子：如果一个时间戳后面时间的差值分别为60,60,59和61，delta-of-delta通过减去前一个时间戳来计算，这样我们得到了0，-1，2。具体算法描述如下：

1. Block的header保存了时间戳t-1（t负1）, 它对齐到两个窗口，t0时间戳保存一个14bit的对于t-1的增量。
2. 对于序列之后的时间戳：
   * 计算the delta of delta：D = (tn −tn−1)−(tn−1 −tn−2)；
   * 如果D为0，则存储单个“0”bit;
   * 如果D在[-63,64]之间，则存储'10'后跟值(7位);
   * 如果D在[-255,256]之间，则存储'110'后面跟着值(9位);
   * 如果D在[-2047,2048]之间，则存储'1110'后跟值(12位);
   * 否则使用32位存储'1111'，后跟D。

在实际的使用中发现96%可以背压缩位单个bit（很显然呀，这么大的QPS，必然很多相邻的时间戳是一样的）。  



#### 数据压缩

 除了数据压缩，对于value也使用了压缩，基于相邻的数据变化不大的特点，主要使用的方法是基于XOR的方法。方法如下描述：
       1. 第一个值是无压缩存储的；
       2. 如果XOR与前一个值为零（相同值），则存储单个“0”bit；
       3. 当XOR不为零时，计算XOR中开头和结尾0的数量：(Control bit ‘0’) 如果有意义的位的块落在先前有意义的位的块内，即至少与前一个值一样多的前导0和多个尾随0，就使用该信息作为  块位置，并且只存储有意义的异或值； (Control bit ‘1’) 否则，用5bit前导零数的长度，然后将有意义的异或值的长度存储在接下来的6bit中，最后存储XOR后值的有意义的bit。

  实际的数据中，大约51％的值被压缩到一个位，因为当前值和以前的值很多是相同的。大约30％的值用控制位'10'压缩，平均压缩大小为26.6位。剩余的19％被压缩为控制位'11'，平均大小为36.9位，这是由于编码前导零位和有意义位的长度需要额外的13位开销。

#### 内存中的数据结构

  Gorilla实现的主要数据结构是一个Timeseries Map（TSmap）。TSmap由时间序列的C++标准库vector<unique_ptr<TSmap>>，和从时间序列名称到的案例保存map组成。该矢量允许通过所有数据进行有效的分页扫描。 使用C++ unqiue_ptr<TSmap>可以在几微秒内复制向量，从而避免影响输入数据流的冗长的关键部分。在删除时间序列时，只标记为‘dead’，但是不会真的删除保存的数据。通过SpnLock的高效实用和分片，以及map有效使用，高效的实现了这个结构。

#### 磁盘上的结构

  Gorilla的一个设计目标就是在单机的故障的下能不影响使用，所以，Gorilla将数据保存在有3个副本的GlusterFS中。 将数据保存在HDFS或其他分布式文件系统也很简单。Gorilla还考虑了单个主机数据库，如MySQL和RocksDB，但是最终没有使用，因为Gorilla不需要数据库查询语言。 一个Gorilla Host拥有多个分片数据，每个分片维护一个目录。每个目录包含四种类型的文件：key列表，追加日志，完整的块文件和检查点文件。
   key列表只是时间序列字符串键到整数ID的简单映射。该整数ID是内存中vector的索引。新keys将附加到当前的密钥列表中，并且Gorilla会定期扫描每个分片的所有keys，以重新写入该文件。数据被流式传输到Gorilla，它们被存储在日志文件中。使用4.1节中描述的格式压缩时间戳和值。但是，每个分片只有一个追加日志，因此分片中的值会跨越时间序列交错，这个增加了存储的开销。Gorilla不会提供ACID保证，因此日志文件不是预写日志。在flush之前，数据通常包含一到两秒钟的数据，最多可达64kB，所以崩溃可能导致少量数据的丢失。
   每两小时，Gorilla将压缩的数据写到磁盘，因为经过了压缩，所以这个比日志文件小得多。这个块文件有两个部分：一组连续64kB的数据块，和一个<time serics ID, data block pointer>的列表。 当一个块文件完成后，Gorilla会更新检查点文件并删除相应的日志。 检查点文件用于标记何时将完整的块文件写入到磁盘。如果在进程崩溃时块文件未成功刷新到磁盘，则当新进程启动时，这个检查点将不存在，因此新进程知道它不能信任该块文件，而应该从日志文件中读取。

### 错误处理

// TODO

## 参考
1. Gorilla: A Fast, Scalable, In-Memory Time Series Database, VLDB '15.
2. https://github.com/facebookincubator/beringei

