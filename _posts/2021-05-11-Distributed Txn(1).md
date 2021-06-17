---
layout: page
title: Distributed Txn With Centralized Sequencer/TSO
tags: [Transaction, Distribution]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## Distributed Transaction With Centralized Sequencer/TSO

### 0x00 Large-scale Incremental Processing Using Distributed Transactions and Notifications

  使用2PC的方式或者是某种2PC的变种方式，与一个中心化的Sequencer，或者是称之为Timestamp Oracle，来作为一个发号器或者是一个全局的时钟，是实现分布式事务的一种方式。很多的这类系统使用的并发控制方式为MVCC，这个中心的全局时钟会和MVCC也强相关。这种方式的一个优点是一些逻辑处理上会比没有中心时钟的要简单，一个缺点是由于事务的进行都需要访问这个中心的时钟，这样系统跨区域部署时候会不太适合。比较。在使用这样的方式的系统中，Percolator这篇Paper是分布式事务设计一般影响比较大一篇，后面很多的设计都是在此的基础上进行变化、改进。Percolator是一种2PC处理分布式事务的方式，其基本的设计核心点有这样的几个：

* Percolator也是利用一个中心话的Oracle来获取时间戳，实现Snapshot Isolation的隔离级别。在读取开始的时候获取一个时间戳称之为start ts，在事务要commit的时候获取一个时间戳称之为commit ts。读取操作读取到start ts为止的最新的版本，如果读取这个版本带上来锁，则要查看这个锁的情况：如果没有超时则需要等待，等待这个锁超时 or 被解除；如果这个锁超时了可能写这个版本的事务没有成功提交，也可能还没有提交。这里读操作会有一个尝试清除这个锁的操作。利用这种方式来解决写事务故障之后没有清除之前写入数据，加的锁的问题。
* Percolator的存储是利用Bigtable，一个分布式的表格系统，Key-Value的value为多列的结构。Percolator利用这个特性， 每个key对应的value又以下这样列：lock列记录锁的情况，对于没有提交的事务这个会记录为primary key的位置，对于primary key记录自己为primary的信息，write列保存已经commited的最大版本，data列记录这个版本对应的数据。另外的notify列和ack_O和事务关系不大。这样Percolator的一个事务的提交与否就是通过primary key来确定来，利用了Bigtable支持单行事务的特点来将其作为一个事务是否commit的一个“开关”，而其它的key就保存这个primary key的信息即可。
* 这样，Percolator的有写操作的事务写的数据都先缓存在本地。事务commit的时候从其中选出一个作为primary，然后写入数据的时候先写入primary，如何写其它的行。每行写之前要检查冲突，基本的方式是检查一行的write列释放存在版本在[start ts , ∞]之间的commited or 活跃的事务，或者是版本在[0, ∞]之间，即任何加锁的版本，如果存在则由于写写冲突无法提交，地abort，基本方式是清除已经prewrite的数据。Prewrite写入的数据为data 和  lock，lock保存的为primary的信息。这样的设计下面，Percolator事务的状态信息，锁信息都是和数据保存在同样的地方。
* 在priwrite完成之后进入commit的流程，基本的方式是尝试提交primary行。由于锁可能被其它的操作清除，在提交之后要检查锁是否已经被清除，通过写入write列数据，这行写入write列的数据对用的start ts，这样就可以找到这行数据start ts的版本并从中读取到数据；并删除lock列即完成提交操作。Primary完成提交之后事务即可以完成commit，其它行数据的提交可以异步进行。而读操作看到这些没有提交的行的时候，可以实际上已经提交了，方式是通过其中记录的primary行的信息去获取对应事务的状态。这样相当于把事务状态的信息就记录到Bigtable之中，没有利用其它的组件。

### 0x01 Omid 的设计

 Omid是在HBase上面做的一个支持分布式事务的系统，其系统类似于前面的Percolator，其实现有很多事类似，但区别还是有不少的。Omid中分布式事务的实现也是使用MVCC，实现Snapshot Isolation的隔离级别。在具体的设计上面，Omid有这样的一些特点：

* 使用中心化的timestamp发号器，称之为transaction status oracle，SO。Omid事务开始的时候，也会去获取一个start ts时间戳，数据读取的时候根据这个start ts来判断数据对本事务的可见性。事务提交的时候，对于写入的每一行，通过SO的lastCommit table中记录的对应key的最后commit的时间戳判断能不能commit。判断的方法就是对应key的last commit小于此事务的starts。所有的行不存在写写冲突的时候事务即可以进入下一部分。这个时候会获取一个commit ts，如何在SO的lastCommit中记录下每个key的lastCommit为这个commit ts。在这里的设计来看， 需要中心化的SO来保存事务的状态信息，本记录事务更新的key最近commit ts的信息。这部分就是将事务状态管理从独立于数据面中的逻辑，和Percolator设计很大的一点不同。

* Omid这里有特点的一点是行写入操作比较简单，之间将key + start ts对应的值设置为要写入的value即可。这样的话，这里写入的时候在事务commit之前，所以这些实际可能是还没有commit的数据，也可能是未能成功commit事务留下的垃圾数据。需要有一种方法来辨识这些数据。对于数据的读取操作， 在读取到小于start ts的数据版本的时候，需要向SO查询这个事务的状态，如果这个事务已经abort了 或者是 还没有commit，这个数据就不应该被这个读取操作读取到。这里也可以看出来Omid设计的一个可能的缺点，就是需要读取的时候也需要一些询问SO的操作。这个是否在这个start ts读取的snapshot中的判断逻辑如下:

  ```
  if Tc(txn_f ) != null then
    return Tc(txn_f ) < Ts (txn_r );
  if T_max < Ts(txn_f ) or aborted(txn_f ) then
    return false;
  return true;
  ```

  这里数据上面保存的为star ts，而释放可见比较的是stat ts和commit ts，所以Omid需要在SO维护一个start ts到commit ts的映射表。这样SO就存在两个table：lastCommit表和committed表。这个在实际中会单机内存的限制，这里需要优化这两个table的空间使用问题。

* 由于Omid使用在中心化的SO中保存lastCommit表和committed表。对于committed表，其优化的思路是删除不会在使用到的事务start ts。SO维护一个T-max，记录了committed表中被删除记录的最大的start ts。这样就需要一种推进T-max的策略。在commit的时候也要主要小于了T-max的表，则只能是abort。另外的一个问题是这之中删除了的记录，后面的事务如何知道其的状态是committed还是aborted呢？为了处理这个问题，Omid又引入了一个记录aborted事务的list。这个aborted list的数据也是要清理的，当前面事务写入的aborted数据被清理之后，SO中abortted list对用的状态数据就可以安全删除了。lastCommit表的规模控制也依赖于T-max，其中小于T-max的记录就可以删除。另外lastCommit表中key记录的是key的hash，这样也可以节省空间。

* 对于数据的可用性，Omid使用的是先有的系统比如HBase，其可用性/可靠性机制也依赖于这样的系统。而Omid新增的SO组件，则依赖于复制，SO使用了WAL的机制来方便一个SO在故障之后启动另外一个SO。这里Log的数据会包括committed、abrted list和T-max的信息。而对于lastCommit表，可以看出其用于的是冲突检测，如果一个SO故障的时候之前所有活动中时候都abort，则后面启动的SO不会和前面的冲突，则就不需要持久化这个lastCommit表了。

上面描述了的Omid的初始版本的设计，后面Omid又发了几篇论文，其中也描述了Omid架构上面的一些变化。比如在后面paper[3]描述的Omid版本的实现中，维护Commit Table变成了直接存储在HBase之上。Commit Table记录txid到commit ts的映射， txid这里就使用start ts。另外没有了lastCommit表，其冲突检测的方式也有一些变化。Omid在FAST ‘17中描述的设计，data table中保存了出来key- value，已经version信息(为txn的txnid)外，还有一个commit的列，记录了commit ts。事务没有提交的时候，对应写入的数据commit列的值为nil，但是事务提交了的情况下，也可能是nil，所以这个时候就需要通过询问TM的commit table来确认一下事务是否commit了。也就是说commit的开关为commit table对应的列，同样记录在HBase中，而不是TM保存。冲突检测的方式变成了：对应write set里面的每一个key，首先为key所在的bucket加上锁，然后检查没有被更大的txid的事务更新。

### 0x02 FoundationDB的设计

 前面的FoundationDB的论文中就可以看出来，FDB在分布式事务的实现方面，和这里谈到的其它的一些方案差别比较大。FDB分布式事务相关的几个设计特点：FDB也是实现的Snapshot Isolation的隔离级别，使用一个中心的Sequencer作为发号器。事务处理的开始由client向一个proxy发送一个请求，获取一个read version，即一个read timestamp。这个read version为proxy从sequencer获取，保证每次获取的version都是递增的。

* 在获取了这个timestamp之后，client可以直接向storage servers发送读区请求。而一个写入的数据会暂存到本地的buffer中。在后面commit的时候，将写入的数据，包括read set和write set的信息，发送到一个proxy。Proxy处理事务提交的请求，如果不能提交，向client返回abort。这里FDB读也是要获取一个start ts，不过读取可以直接读存储节点。冲突检测在实际commit之前，有单独的一个组件检测；

* FDB commit的流程：1. 从sequencer获取一个commit version；2. 然后，proxy将事务commit相关的数据发送给resolver。在resolver返回没有冲突的情况下，进入下一步，否则返回abort；3.  事务可以commit的时候，proxy将数据发送个LS来持久化数据。在所有指定的保存这个的log复制返回成功之后，proxy向sequencer报告commit version的信息，然后恢复client。SS会从LS拉去log，应用到本地形成最终的数据。这里FBD的写log的部分和实际存储的部分是分离的，和一般在同样的节点；
* FBD就没有和其它类似于Percolator那样的2PC的逻辑，事务的commit主要步骤为：proxy获取commit的数据，获取commit ts；如何检测冲突；后面写入LS进行数据持久化。这几步完成即可以认为committed。另外Paper[5]描述的事务流程和能够找到的早起的FDB事务实现的一些方式存在一些差别。

### 0x03 其它类Percolator的设计

TiDB基础版本的分布式事务实现方式上，基本上使用了Percolator的方式，不过后面在基础的方式上面做了很多的优化。在TiDB基础的设计中，其：1. 也是使用start ts和commit ts两个时间戳，时间戳的分配从中心化的TSO分配，数据写入的时候会暂时buffer早事务本地；2. 事务commit的时候，选取其中一个作为primary key，然后prewrite数据，底层的分布式KV收到请求之后检查版本冲突等，通过之后prewrite数据并加锁；3. 在所有的prewrite成功之后获取一个commit ts，发出提交primary key的操作。Primary key提交成功即事务committed。这样的设计基本上延续了Percolatord的设计，不使用中心化的transaction manager，一样的2PC实现方式。

* 在TiDB基本的设计上，其缺点是网络交互较多、存在中心化的TSO服务、以及数据buffer事务本地可能对一些大事务不友好。后面其加入了一些优化：比如悲观事务，这里悲观事务和乐观事务的区别在于：在事务prewrite之前，增加了一个Acquire Pessimistic Lock 的阶段。每次数据修改的操作都会加上锁，锁也是保存到底层的分布式KV中。在加锁之后，写写之间也相互conflict/阻塞的的，但是写读不会。事务后面commit的操作基本和之前相同，这里需要保证在加锁之后，不会出现新的conflict。为了处理这个悲观事务的锁，加入了一个Waiter Manager，一个悲观事务遇到其它事务加的锁时，需要通过Waiter Manager来等待这个锁被释放。这里本质上就是将在prewrite阶段的pwrite数据并加锁的操作中加锁的操作提前到prewrite之前。
* 通过加锁的方式的话，需要处理的一个问题就是死锁的问题。TiDB死锁检测的基本方式是在下层的分布式KV中，选择一个节点作为deadlock detection的leader。这个节点上面会记录并检查正在执行事务的依赖关系。在一个事务要进入等待锁的操作机会，需要请求这个节点来检查deadlock。这个检查可以是异步的，在返回可能存在死锁的时候不继续等待即可。

  处理在延迟上的一些优化外，TiDB之前还引入了对Percolator 2PC方式对大事务的一些优化。Percolator的方式，prewrite的数据会阻塞后面的操作，而且容易超时被其它的事务清理掉。对这些确定优化的基本思路是：1. 对事务进行重新排序，基本的方式是在priamry key保存的数据中，记录下一个min commit ts。这个min commit ts第一步开始时能改选择的最低的commit ts。如果一个读事务，从primary key保存的min commi ts发现，一个进行中的更小的事务，则可以选择增大了min commit ts，来强行将这个事务推迟到后面，来避免对当前读事务的阻塞；2.  另外一个优化点是锁的过期时间，这里引入了一个动态更新TTL的机制，来避免一些进行中的事务被清理掉的情况。

## 参考

1. Large-scale Incremental Processing Using Distributed Transactions and Notifications, OSDI '10.
2. Omid: Lock-free Transactional Support for Distributed Data Stores, ICDE '14.
3. Omid, Reloaded: Scalable and Highly-Available Transaction Processing, FAST '17.
4. Taking Omid to the Clouds: Fast, Scalable Transactions for Real-Time Cloud Analytics, VLDB '18.
5. FoundationDB: A Distributed Unbundled Transactional Key Value Store, SIGMOD '21.
6. https://docs.pingcap.com/zh/tidb/stable/optimistic-transaction, TiDB乐观事务.
7. https://book.tidb.io/session1/chapter6/pessimistic-txn.html, TiDB悲观事务.

