---
layout: page
title: CockroachDB - The Resilient Geo-Distributed SQL Database
tags: [Database, Transaction, Distributed]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## CockroachDB: The Resilient Geo-Distributed SQL Database

### 0x00 事务

与上面Taurus Database的Paper相比，这篇Paper描述的CockroachDB数据库的架构师模仿Spanner的架构。Paper的内容没有太多新的内容，因为CockroachDB主要的技术都是已经被网上有了相当多的描述，这篇Paper类似于一个总结。看这篇Paper需要对CockroachDB(CRDB)已经有了一些的了解。事务这里首先就设计到了在CRDB中事务操作的基本逻辑和写入事务两个核心的优化。CRDB中，一个事务从一个gateway节点接受到SQL请求，这个节点会负责和客户端进行交互，另外有作为一个 transaction coordinator 。对于一个coordinator事务相关的部分操作逻辑如下。这个描述地很简洁，一些细节方面就里面设计到很多的，具体可以参考网络上面能够找到的资料。这里按照Paper中的描述来说明一些事物操作的基本流程已经一些主要的优化。从coordinator的角度来看，这里已经是SQL转化为来若干的KV层面的请求。一般的逻辑下面，下一个操作的发出得等到上一个操作发挥，另外就是操作得复制到足够的节点上面才能认为是完成了。为了优化这两个问题，CRDB引入的优化就是Write Pipelining 和 Parallel Commits：Write Pipelining可以使得一个操作在没有复制完成之前就返回，而 Parallel Commits使得commit操作可以和write pipeline replicate操作不符进行。

* Write Pipelining。这里讨论的一般的事务情况形如这样的操作:

  ```
  BeginTxn;
  Operation-0;
  Operation-1;
  Operation-2;
  CommitTxn;
  ```

  不同的操作一般的情况下就是先后执行。最直接的方式是每次操作都成功之后在执行下一个操作，这样的方式性能方面会有比较大的问题。另外的实现方式，和很多相关的系统使用的一种思路是将写入操作的数据先保存到本地，然后在Commit的时候一起去写入，这样避免了一些分布式的操作。而这里使用的方式是Write Pipelining，即每个操作可以直接返回结构，然后异步的执行操作。在Commit的时候再去评估一个操作是否成功，这个也利用了这个BeginTxn然后在执行操作最后执行Commit操作上面的特点。所以在这种情况下，Coordinator需要知道目前正在操作中的操作，记录在inflightOps中。CRDB使用MVCC的方式，这里在开始的时候也是获取了一个时间戳。这里的时间戳也是从本地获取的，和一些类似的系统使用一个中心化的时间戳分配服务不同，也和CRDB主要参考的Spanner的TrueTime有所不同，不需要特殊的硬件支持。一个操作在不是commit操作的情况下，且和前面的操作没有重叠，这个操作就可以pipeline执行。如果一个操作和前面的操作有重叠，就需要等待前面的操作被复制完成，这个称之为pipeline stall。实际的操作被发送到Leaseholder来执行。这里注意的是Leaseholder操作可能返回一个更大的时间戳ts，这种情况下，需要检查对应的key在ts到返回的ts中间的这段时间也没有改动，如果没有更新这个txnTs，如果有这个事务就被认为是操作失败，可能需要重试操作。

* Parallel Commits操作则使得commit的操作可以异步执行。体现在下面算法中的最后两步。一个如果一个事务的所有写入内容都被复制完成之后，后面的commit操作是一定能够完成的。所以到了这里就可以先返回成功再去实际地commit。为了实现这个功能，CRDB引入了一个staging的事务状态，表示一个事务操作的所有写入是否都复制完成了。这个优化取得的效果很明显，

  ```
  Parallel Commits improves throughput by up to 72% and reduces p50 latency by up to 47% when the table has one or more secondary indexes, since index updates require multi-Range transactions. 
  ```

![](/assets/png/crdb-txn-cor.png)

对于Leaseholder，接受到一个请求是先校验自己的lease，然后获取需要的latch。并等待起以来的操作的复制操作完成。低于非只读的操作，还需要更新op.ts，这里设计到CRDB另外的一些逻辑。执行操作完成之后，对于非commit操作执行可以结果。而后面对于非只读的操作，在去复制操作和将操作apply到本地的存储引擎。之后释放latch。需要的时候返回commit的结果。

![](/assets/png/crdb-leaseholder.png)

 写入的原子性则是通过write intents来实现，和Percolator的一些思路类似。Wriite intent是一些额外元数据信息的KV Pair数据。额外的元数据中，会带有执行这条记录相关的事务的transaction record的信息。通过transaction record可以获取到这个事务的状态信息：是pending, staging, committed 还是 aborted。读取操作的时候，遇到一个intent的记录时，需要去读取这条记录的transaction record。如果是commited状态，则当中是一个正常的记录来处理，并删除intent相关的元数据。如果事务是aborted的，这条记录会被忽略并删除这条记录。如果状态是pending的，则需要等待写入操作的完成。如果是staging的状态，则这个不能确定事务是committed还是aborted，需要额外的操作来确认状态。如尝试阻止对应事务的一个写来abort掉这个事务，如果写入操作已经是复制完成了，则对应的事务则相当于提交了。CRDB的Concurrency Control使用的是MVCC的方式，这里还有一些有意思的ts调整的逻辑。CRDB中一个事务执行的时候，和一些使用start-ts和commit-ts两个时间戳的系统不同，CRDB只使用了一个ts。并发冲突的一些处理逻辑：

* Write-read conflicts，即一个读请求读取到了一个ts更小的还没有commit的数据，这个时候需要等待对应的事务完成操作，commit or abort。这里等待使用一个内存中的队列来实现。而读取到ts更大没有提交的的数据的时候，就直接忽略这些数据；

* Read-write conflicts，一个事务准备去写一个key的时候，如果这个key已经被有一个不小于这个事务的ts的事务读取的时候，就不能操作。这里使用的方式是调整这个事务的 commit timestamp为读取事务的ts。对应到上面的ts调整逻辑；

* Write-write conflicts，一个写入操作遇到一个没有提交的ts更新的记录的时候，需要等待对应的事务操作完成，commit or abort。如果遇到了一个更高时间戳的记录，则和read-write conflicts调整时间戳。这里还需要处理deadlock的问题。不过这个分布式情况下的死锁检测也该也是一个比较麻烦的事情。

  ```
  Write-write conflicts may also lead to deadlocks in cases where different transactions have written intents in differ- ent orders. CRDB employs a distributed deadlock-detection algorithm to abort one transaction from a cycle of waiters.
  ```

这里的Read Refreshes比较有意思。前面提到的冲突处理的时候会有调整commit ts的逻辑，但是这里需要处理另外的一些问题。调整一个事务的read timestamp Ta到一个更大的ts Tb，需要保证在(ta,tb]这个区间内，读取的数据没有被更新。这里是应该为了实现可重复读，满足这里隔离级别的要求。如果发生了更新的情况，则可能进行重试 。对于交互性的事务如果返回了一部分结果给客户端的，则不能重试。为了处理读取这些的keys的事务的信息，这里使用的方式维护一个事务的read set。一个łread refresh请求需要验证在给定的时间区间内没有被更新。这里一个写入事务是如何知道读取事务的信息的，查了一些资料是在leaseholder中的锁逻辑中处理的。

#### Follower Reads

 CRDB使用Follower Reads来实现一些只读事务的操作，需要在SQL加上一些指示性的东西。满足Follower Reads要满足这样的一些条件，一个是对于一个时间戳为ts的事务，他可以读取的数据不会在后面被更新，另外一个就是要去这个non-leaseholder replica已经有了这些数据。也就是要去满足non-leaseholder replica执行一个时间戳为ts的只读事务的时候，leaseholder不能在执行比这个ts还还要小的写入操作，另外就是这个non-leaseholder replica 的raft log的prefix要足够支持执行这个事务。为了满足这些条件，每个leaseholder会记录下进行中的操作，并周期性地通知一个closed timestamp，表示不会在接受小于这个closed timestamp的写入事务，

```
  Closed timestamps, alongside Raft log indexes at the time, are exchanged periodically between replicas. Fol- lower replicas use the state built up from received updates to determine if they have all the data needed to serve consistent reads at a given timestamp. For efficiency reasons the closed timestamp and the corresponding log indexes are generated at the node level (as opposed to the Range level).
```

### 0x01 Clock Synchronization

 时钟是一个处理起来很麻烦的东西。这里有几种不同的处理方式，一种是这种架构开创的Spanner使用TrueTime来解决，需要特殊的硬件，而另外的一种就是使用一个中心化的时间戳服务，而CRDB使用的是基于HLC的方式，但是可以的误差比较大，在几百毫秒级别。使用HLC这样的方式优点是方便支持跨区域分布，而使用中心化的时间戳服务的时候就不太好跨区域。HLC这里总结了几个特点，对于实现满足一直想要求的比较重要：一个是causality tracking，即可以从中获取causality的内容，HLC的时间戳信息会带在节点之间通信的消息当中。这个特性对于实现for each Range, each lease interval is disjoint from every other lease interval是必要的。第二个特性是strict monotonicity，从跨节点还是单节点，在重启的情况也是，这个显而易见必须满足的。另外一个是self-stabilization，在单个的物理时钟有问题的时候也可以稳定下来(This provides no strong guarantees but can mask clock synchronization errors in practice.)。

#### Uncertainty Intervals

使用这种方式的时候，会遇到的一个问题是Uncertainty Intervals。也就是比较时间戳先后的时候，由于不同机器的时间是有差别的，会有一个无法比较的时间interval。CRDB支持的隔离级别是serializable，数据库系统中的serializability对一个事务顺序和其对应的时间顺序是没有任何要求的，指的是执行看上去的效果是serializable就行。一般情况下CRDB对于单个key满足Linearizability（Under normal conditions, CRDB satisfies single-key linearizability for reads and writes.）Paper中的描述加上了一个Under normal conditions。使用 loosely synchronized clocks的时候也可以的，要求始终偏差要在一个maximum clock offset之内。对于涉及不相交的key set的时候，不同事务部满足strict serializability。事务执行获取commit ts的时候，这里会得到一个[commit_ts, commit_ts + max_offset]的不确定区间。事务执行的时候，如果遇到了不在这个区间内的，正常安装这里表示出的先后顺序来操作即可。如果在这个区间之内，CRDB的处理方式是执行一个uncertainty restart操作，将commit timestamp增大为比不确定区间upper bound还要大的一个值。

#### Behavior under Clock Skew

 前面操作的正确性依赖于clock的偏差在一个范围之内。一个Range内使用Raft维护consistency的，时钟的偏斜不会对这个造成影响。但是时钟偏斜可以不同的节点认为自己有这个range的lease（However, Range leases allow reads to be served from a leaseholder without going through Raft. This causes complications because under sufficient clock skew, it is possible for multiple nodes to think they each hold the lease for a given Range.)。CRDB引入两个保证来避免出现这些问题：一个是range lease包含了一个start和end的时间戳，leaseholder只能在这个范围内服务。一个range的不同的lease是没有相交范围的。另外一个是一个 Range’s Raft log的写入会带上range lease的序列号。复制成功的时候，这个序列号被再次检查，是否对得上currently active lease。如果对不上写入会被拒绝，

```
  The first safeguard ensures that an incoming leaseholder cannot serve a write that invalidates a read served by an outgoing leaseholder. The second safeguard ensures that an outgoing leaseholder cannot serve a write that invalidates a read or a write served by an incoming leaseholder. 
```

 在不能保证这个max offset在一个区间内的，CRDB的一些操作的正确性就不能保证。为了处理可能的这种情况，CRDB会周期性地检查 clock’s offset，超过配置的最大值的80%的时候，会让一个节点self-terminates的方式来处理。

### 0x02 SQL & Lessons Learned

 SQL层这里感觉不是这篇paper描述的重点。CRBD的SQL Data Model就是一般关系型的SQL Data Model。查询优化其使用Query Optimizer风格的优化器。Query transformations使用一种称之为Optgen的DSL。对于优化器可以感知地理分布，比如一个查询`SELECT * FROM t WHERE id = 5` 可以被重写为 `SELECT * FROM t WHERE id = 5 AND (region = ’east’ OR region = ’west’)`，来方便使用index跳过一些查询操作。在执行模式上面，可以有gateway-only mode和distributed mode，第一种只会有产生查询计划的节点处理这个SQL的操作，而后面一种会多节点处理。后一种只会应用到read-only的操作情况下面。执行引擎支持Volcano风格的迭代的模型，一次处理一行的数据row-at-a-time，也可以使用一个vectorized execution engine，每次处理一批的数据。Schema Changes使用F1使用的思路等等。另外在Paper中提及到了几个经验：

* Raft Made Live，Raft层面的优化：1. Reducing the Chatter。Raft的leader使用周期性的heaetbeat来维护自己的leadership。在一个很大的CRDB的情况下，会有非常多的Raft Group。Heartbeat带来的开销也是很大的，CRDB使用的优化方式是：1. 将同个节点不同group的rpc融合为一个；2. 如果一个group最近没有接受到写入，则可以暂停这个heartbeat。2. Joint Consensus，原来的Raft的membership change只支持一次处理一个，这个会早rebalancing operations 操作的时候造成一些问题，比如移动一个副本到另外一个节点的时候，可能造成(1) temporarily dropping down to two replicas, 或者是(2) temporarily increasing to four replicas, with two in one region。这里引入称之为 Joint Consensus来解决这个问题，这里没有提及到具体的方法。
* 其它一些问题。

### 0x03 评估

 这里的具体信息可以参看[1].

## 参考

1. CockroachDB: The Resilient Geo-Distributed SQL Database, SIGMOD '20.

