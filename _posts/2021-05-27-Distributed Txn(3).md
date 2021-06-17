---
layout: page
title: Distributed Transaction from Research Papers
tags: [Transaction, Distribution]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## Unifying Timestamp with Transaction Ordering for MVCC with Decentralized Scalar Timestamp

### 0x00 基本思路

  这篇paper是关于不使用中心化的TOS服务来实现分布式的MVCC的一种方式。其思路通过前面的一些方式的优化之后其实就可以得出一个大概。Paper是提出了一种不使用vector timestamp，也不使用中心化的TOS的MVCC的实现方式。Paper中描述了使用中心化的TSO实现MVCC的基本思路。其方式前面谈到的使用一个中心化transaction manager实现的思路是一样的，下面是用这篇paper的语言来描述。

```
// 2PL with global timestamp (GTS)
At Oracle: GlobalTS, StableGTS, Queue // monotonic global timestamp, snapshot global timestamp, pending global timestamp
INSTALL(gts):
  add gts to Queue
STABILIZE(): // run asynchronously
  for each gts in Queue do
    if gts is ready then
      dequeue gts and gts → StableGTS
 
At Worker i: // i denotes the worker number WRITE(tx, id, data)
  acquire lock
  add <id,data> to tx.wset

READ(tx, id)
  acquire lock and get latest <data>
  add <id, data> to tx.rset
  return <data>

COMMIT(tx)
  tx.TS ← Oracle.GlobalTS // network round trip
  for each w in tx.wset do
    update <w.data, tx.TS> and release lock
  for each r in tx.rset do
    release lock
  Oracle.INSTALL(tx.TS)

ROTX(tx)
  tx.TS ← Oracle.StableGTS
  for each r in tx.rset do
    get <r.data> up to tx.TS
```

另外这里还描述了一下使用Vector timestamp (VTS)的来实现的方式。下面描述的这里还是存在一个中心的Orcale，用于记录stable vector timestamp，其麻烦的一个地方在于其要维护一个事务的依赖，通过在Oracle中记录已经完成的事务，来得出stable VTS的信息。

```
At Oracle: StableVTS,  Queues //snapshot global timestamp, pending global timestamp

INSTALL(<i:ts>, deps)
  add <<i:ts>, deps> to a Queues[i]

STABILIZE() //run asynchronously 
  for each queue in Queues do
    for each <<i:ts>,deps> in queue do
      if deps is ready then // stability protocol
        dequeue <<i:ts>,deps> 
        <i:ts> → StableVTS
        
At Workeri: // i denotes the worker number 
 LocalTS //monotonic local timestamp

WRITE(tx, id, data)
  acquire lock and get latest <i:ts>
  add <id, data> to tx.wset
  add <i:ts> to tx.deps

READ(tx, id)
  acquire lock and get latest <data, <i:ts>>
  add <id, data> to tx.rset
  add <i:ts> to tx.deps
  return <data>
 
COMMIT(tx)
  tx.TS ← LocalTS
  for each w in tx.wset do
    update <w.data, <i:tx.TS>> and release lock
  for each r in tx.rset do
    release lock 
  Oracle.INSTALL(<i:tx.TS>, tx.deps)

ROTX(tx)
  tx.TS ← Oracle.StableVTS
  for each r in tx.rset do
    get <r.data> up to tx.TS
```

 这里提出的思路Decentralized Scalar Timestamp (DST)。使用DST执行读写事务的基本思路如下。其这样的设计处于这样的一些考虑，首先是先不使用中心化的TSO，每个节点都是使用本地的时间戳。另外考虑：假设一个事务A在另外一个事务B之前commit，它们在访问了同一个数据的情况下，则：1. 两者都是写入的情况，最后留下的 or 更新版本的数据是B写入的；2. A Write 而 B Read的情况下，B要能读取到A的写入；3.  A Read 而 B Write的时候，A不能看到B的写入。从下图的示意代码来看，DST有这样的一些特点：

* 读取数据 or 写入数据的时候，都会更新本事务的时间戳，记录为当前写入、读取过的数据的最大的timestamp+1。这里读取和写入都会加上锁，另外读取的时候读取最后一个版本的数据，并获取其timestamp的信息。这样可以保证这个事务commit的时候，没有更新版本的数据写入。这里每次都更新了事务的timestamp情况下，也能够保证其都是到的是一个snapshot。
* 在事务commit的时候，每个写入的数据的timestamp会记录为到commit时候事务记录的timestmap，也就是之前读取、写入过的数据的time timestamp + 1。写数据更新数据ts可以保证后commit的事务一定是有更大的timestamp，保证来第1点。另外读取的时候会更新事务的ts，使其成为比前面已经写入数据对应事务更晚的事务，因为会使得当前的事务有更大的timestamp，保证来第2点。另外一个释放read set里面锁的时候的时候，也会更新数据的ts，后面事务写入的时候会使得其后面写入这个数据的事务有更大的timestamp，从而满足第3点。
* 最后将本地的local ts更新为事务的目前的timestamp。这样能保证后面的事务来的时候能读取到前面committed的数据。可以看出来DST的一个思路是除非发挥一个事务的ts是可以满足一定条件下，是可以调整的特点。DST这里读取数据的时候也会涉及到数据的更新，在实际的系统中这个很容易成为一个问题。而且读写之间也是相互阻塞的。另外从下面的只读事务来看，好想读写事务中的读操作和也是相互阻塞的？读写事务中的读操作和只读事务的读操作也是阻塞的？如果是这样的话那基本.....

```
At Worker i: // i denotes the worker number 
  LocalTS // monotonic local timestamp

START(x)
  x.TS ← LocalTS

WRITE(x, id, data)
  acquire lock and get ts
  add <id, data> to x.wset 
  x.TS ← max(x.TS, ts+1)
  
READ(x, id)
  acquire lock and get latest <data,ts>
  add <id, data> to x.rset 
  x.TS ← max(x.TS, ts+1)
  return data

COMMIT(x)
 for each w in x.wset do
   update <w.data,x.TS> and release lock
 for each r in x.rset do
   update <x.TS> and release lock
 LocalTS ← max(LocalTS, x.TS)
 
ROTX(x)
  x.TS ← LocalTS
  for each r in x.rset do
     DEP_READ(x, r)

DEP_READ(x, r)
  if r.ts <= x.TS then
     r.ts ← x.TS
     wait until r not locked 
   get <r.data> up to x.TS
```

另外，对应只读类型的事务，这里也会涉及到数据的更新，为更新第timestamp，避免出现后面的写入事务时间戳更小的情况。从这里也可以看出来工业风格的paper和学术风格的paper存在的差异，这篇这样风格的基本上就从一个idea出发弄出来一篇，看上去缺乏一些实际的可行性？

## Clock-SI: Snapshot Isolation for Partitioned Data Stores Using Loosely Synchronized Clocks

### 0x10 基本思路

  Clock-SI这篇在前面已经看过了[3]，其设计也影响了后面的一些系统的设计。Clock-SI不使用中心化的TOS，而不是每个partition使用本地的local ts。时间戳同样适用start ts和commit ts两个。数据读取的时候，先从本地获取一个start ts，然后取读取不同分区的数据。读取这里特点就读取操作可能因为partition的local ts小于事务的start ts而被阻塞，要求等等partition的local ts过了start ts才行。另外对于读取的数据，更新这些数据的事务如果处理prepare or committing的状态，也需要等待其到committed or aborted。

```
StartTransaction(transaction T) 
  // Clock-SI直接使用本地时钟，在需要的时候可以减去一个∆，
  // 来保证读取的成功率or读取比较旧的数据
  T.SnapshotTime ← GetClockTime() − ∆ 
  T.State ← active
 
ReadDataItem(transaction T, data item oid)
   if oid ∈ T.WriteSet return T.WriteSet[oid]
   // check if delay needed due to pending commit 
   // 解决前面时钟的问题和Pending事务的问题就是开等待
   if oid is updated by T′ ∧ 
      T′.State = committing ∧ 
      T.SnapshotTime > T′.CommitTime 
   then wait until T′.State = committed 
   
   if oid is updated by T′ ∧ T′.State = prepared ∧
      T.SnapshotTime > T′.PrepareTime ∧
       // Here T can obtain commit timestamp of T′ 
       // from its originating partition by a RPC. 
       T.SnapshotTime > T′.CommitTime
    then wait until T′.State = committed
    return latest version of oid created before T.SnapshotTime
    
upon transaction T arriving from a remote partition
    // check if delay needed due to clock skew
    if T.SnapshotTime > GetClockTime()
    then wait until T.SnapshotTime < GetClockTime()
```

而对于事务的commit来说，Clock SI的commit ts使用在prepare阶段从各个参与的partition中获取到的最大的timestamp。事务提交也是使用的2PC的方式。这样就能过保证后面commit的事务的commit ts一定会大于之前读事务、读写事务的timestamp。这里读取的话，读取是能过保证读取到一个snapshot，满足snapshot isolation的一些要求。但是Clock SI不能满足一定能过读取到最新的已经committed的数据。

## ConfluxDB: Multi-Master Replication for Partitioned Snapshot Isolation Databases

### 0x20 事务的基本思路

 这篇paper描述了ConfluxDB关于分布式事务和复制的一些设计，这里提出的方法称之为Collaborative Global Snapshot Isolation (CGSI)，这个CGSI不需要中心化的timestamp服务，和Clock-SI不同的其也不需要一个实际的clock，而只需要的一个*event clock* ，这实际上是一个logical clock。在描述这个方法之前，先定义了这样的一些概念: 1. dependent(Ti, Tj )表示在一个参与者中，事务Ti的读写集合和事务Tj的读写集合有交叉，不包含读读的交叉；2. 一个事务Ti看到的是一个一致的snapshot，要满足在两个不同sites s, t执行的事务Ti Tj，不能有`dependent(Ti,Tj) ∧ (Ti-s ≺ Tj-s)∧(Ti-t ⊀Tj-t)`。即一个事务另外一个事务的影响，不能部分可见。只能要么都不可见，要么都可见。3. 对于已经提交 or 正在运行中的事务，其都氛围2类: concurrent-s(Ti)表示在site 上执行的一些事务，这些事务begin(Ti-s) ≺ commit(Tj-s)，没有满足这个表示为serial-s(Ti) 。整体来说，有这样的几个要求：

```
I1. If dependent(Ti,Tj) ∧ (begin(Ti-s) ≺ commit(Tj-s)), then begin(Ti-t) ≺ commit(Tj-t) at all participating sites t.
I2. If dependent(Ti,Tj) ∧ (commit(Ti-s) ≺ begin(Tj-s)), then commit(Ti-t) ≺ begin(Tj-t) at all participating sites t.
I3. If commit(Ti-s) ≺ commit(Tj-s), then commit(Ti-t) ≺ commit(Tj-t ) at all participating sites t.
```

 这几个描述的主要是在有dependent关系的事务之间，其begin-ts被commit ts必须满足一定的顺序。对于ConfluxDB，其Distributed Certification算法如下：

```
1. 协调者coord-i请求site s内的参与者，获取concurrent-s(Ti)的事务列表，参与者收到这个请求是返回它的concurrents(Ti)的数据；
2. 协调者收到参与者回复之后，将所有的列表合并为一个gConcurrent(Ti)，如果存在一个事务Tj在这个列表里面，则Tj和Ti至少在一个site里面为并发执行的关系；
3. 后面协调者就需要吧这个gConcurrent发送给所有的sites，每个site检查吃饭存在这样一个事务，其满足dependents(Ti,Tj) ∧ (Tj ∈ gConcurrent(Ti)) ∧ (Tj ∈ serials(Ti))。存在则可能不能满足consistent snapshot的要求，要返回一个错误，否则返回正确；
4. 协调者在收到sites的回复，如果存在错误，则事务得abort，否则可以继续。
```

 ConfluexDB事务执行也基本上使用了2PC的方式。其为体现事务之间commit的关系:

* 每个site维护一个递增的event clock ，协调者在给participants发送prepare消息的时候，会带上这个event clock的值；2. Participants在收到这个消息之后，更新本地的event lock为其见过的event clock的最大值，然后给coordinator返回的消息里面带上这个更新之后的值。
*  在收到所有的participant返回的prepare ok消息之后，根据返回的数据更新自己的event clock为更大的值。在收到的所有的回复之后，自增自己的event clock，然后将当前的event clok值作为commit ts；4. participant在收到commit的消息之后，根据commit消息里面带上的event clokc值，更新自己的event clock为更大的值。对于方法的正确性证明，paper[4]中有描述。

 从这里来看ConfluexDB保证读取到一个一致的snapshot的是在写事务commit的时候，检查是否其commit会影响snapshot的一致性，如果不满足则需要abort可能导致不一致的事务。这样是否会则成一些读请求会导致一些写操作无法commit呢?

## 参考

1. Unifying Timestamp with Transaction Ordering for MVCC with Decentralized Scalar Timestamp, NSDI '21.
2. Clock-SI: Snapshot Isolation for Partitioned Data Stores Using Loosely Synchronized Clocks, SRDS ‘13.
3. https://nan01ab.github.io/2019/02/SIs.html, Clokc-SI.
4. ConfluxDB: Multi-Master Replication for Partitioned Snapshot Isolation Databases, VLDB '14.

