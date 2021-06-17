---
layout: page
title: Optimzations of Distributed Txn With TSO
tags: [Transaction, Distribution]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## Optimzations of Distributed Transaction With TSO

### 0x10 TiDB Async Commit

  前面提到TiDB使用的方式基本沿用了Percolator的方式，这种方式的一个特点是网络交互次数较多。Async Commit顾名思义就是将2PC中，第2步的commit操作异步话，意味着返回给client就prepare完成即可以了。Percolator的方式中，一个事务的状态有primary key确定，如果要改成异步commit的话，意味着需要在prepare这一步完成之后，就可以知道事务的状态。考虑一个prepare返回协调者就crash的情况：一种情况是协调者知道了事务可以正常commit，但是没来得及commit，也可能参与者都返回了ok但是还没有收到就crash；另外一种情况是协调者收到了不能commit的信息，但是也没有来得及abort。后面其它的操作看到了这个事务遗留下来的信息，需要可以从其中恢复出来，这样就需要：

* 这个事务是可以commit还是不可以commit。一个操作在看到一个状态不确定的数据时候，就需要知道对应事务所有写入数据的状态。由于之前在非primary key上面保存了primary key，这样继续在primary key的信息里面，保存上这次事务其它的keys，就可以根据这个信息取一一询问。这里可以看出这种方式适合OLTP的应用场景，对大事务不是很适合；
* 可以commit的时候，和这个事务commit ts是什么。这里对commit ts有两个要求：一个是这个commit ts必须体现事务之间的commit的顺序，另外一个是这个commit ts不能太大，因为后面读操作的时候能改立即读取到committed的数据，又读取操作获取的start ts是从TSO而来，这样要求commit ts不能大于TSO已经分配的ts+1。为了实现这些要求，这里在TiKV上维护了一个max ts，另外在prewrite操作的时候，这个请求携带了一个从TOS获取的prewrite ts。KV的节点在收到请求之后，如果本地的max ts小于这个prewrite ts，需要增大这个ts。KV节点持久化prewrite的数据，并记录下min commit ts为max ts。KV节点给DB节点返回的时候，会带上这个min commit ts。事务commit的时候，选择commits为min commit ts中最大的一个。
* 对于只读的事务，其操作也会导致KV节点上面的max ts 更新。KV节点接受到只读操作的时候，会根据请求带上的start ts更新增大max ts。这样后面写操作选择的一个min commit ts至少为max ts +1。这样后面的事务commit之后，不会破坏前面的snapshot读操作的snapshot语义。引入了Async Commit之后，如果一个事务知涉及到了一个KV节点，就可以使用1PC的方式。Commi ts用前面同样的方式得出。
* 另外一个在Async Commit的设计文档中，谈到了其和副本读的一些兼容性。因为现在commit ts可能从KV节点得出(虽然最终的来源还是TSO)，而且读操作也会推大节点上的max ts。为例保证读的snapshot语义，需要副本也感知到这些变化，读副本的时候也需要更新这个max ts。兼容的基本方式就是在对leader节点的一个ReadIndex RPC请求中加入了读请求的start ts，以及带上读取对应的range(s)信息。

#### 另外一些问题

这里在引入了这个之后其实会引入另外一些问题需要处理。比如有一个问题：考虑一个事务在prewrite阶段，这个事务写入的一部分keys，包括primary key在内，已经完成prewrite且返回成功，另外一部分keys prewrite的操作由于某些原因，发出请求的动作被卡住，or 请求消息由于网络的原因 or 其它什么的原因这个请求还没有被对应的节点处理。这个时候另外一个事务的操作，在看到了这个事务写入的一部分keys之后，通过查看primary key获取写入的key list，发现不全。

* 这种情况下发现不全的事务如果认为是这个之前事务不能prewrite成功(因为数据缺失了)，且这个锁过期了，这个事务如果将之前事务写入的部分清理掉。而后面等待之前事务剩余的部分keys写入成功，就获取到了全部keys prewrite成功的消息，从而返回给client成功。这样会导致无法commit事务返回了可以commit的情况。因为这里一个事务看到一部分写入内容的时候，它是无法区分是之前写入这些数据的事务不能成功prewrite，还是prewrite还没有完成。如果直接选择不清理这些数据，则可能造成确实无法commit的操作一直阻塞后面的操作。所以只能选择在满足条件的情况下，清理这些部分写入的数据，比如在锁/事务超时的情况下。选择了这样的化，就必须要处理prewrite的数据已经有被清理的情况，这里一个解决办法可以是记录一个max aborted ts，即记录abort了的事务可能选择的commit的最大的ts。如果后面有prewrite小于这个ts的操作，必须得返回拒绝。

另外在文档[3]中，记录了另外几个需要处理的问题。比如One-phase Writing Locking 问题，比如globally-non-unique-timestamps的问题。第一个问题因为是因为读事务也会推大KV节点的max ts。在文档[3]中描述的commits ts最终为max_over_all_keys{max_read_ts, start_ts, _for_update_ts}+1。这里的max read ts和 for update ts应该就是max ts的来源。获取了max read ts，这个消息的作为prewrite的一部分数据被写入，以便于后面可以恢复计算出commit ts。但是写入操作的时候，又可能发生了其它的读取推大了这个max read ts。这里就需要一种机制来避免在写入的过程中，这个ts被改变。基本的解决思路是加上一个锁。globally-non-unique-timestamps的问题KV节点需要使用回滚的操作得记录一个rollback的记录，使用`{user_key}{start_ts}`作为key，而commit的记录使用`{user_key}{commit_ts}`作为key，在使用了Async Commit之后，这两个中的start ts和commit ts可能是相同的，解决这个基本的思路是在需要的时候记录一个flag的方式。

### 0x11 Taking Omid to the Clouds: Fast, Scalable Transactions for Real-Time Cloud Analytics

 这篇paper[4]是Omid发表关于其优化思路的一篇。这里主要是提出了2中的事务优化方式。一种方式称之为Low Latency Transactions，简称Omid LL，其优化是将Commit Table的更新移动到client端来操作，简化Transaction Manager的逻辑；另外一个优化称之为Fast-Path Transactions，其优化是对于单个key操作的优化，将其优化为1PC的方式。对于第一个优化点：

* 在使用了Omid LL之后，TM的操作就只有了分配时间戳和冲突检测的功能。事务开始的时候请求TM获取一个start ts，数据写入的时候，写入记录start ts，而commit ts相关的列记为nil。事务在后面commit的时候，向TM发送一个commit的请求，TM会检查这个事务的冲突情况，并返回一个commit ts。而对于client端而言，对于client commit操作的改变，主要是要添加在请求TM commit之后，TM如果返回可以commit，则client加入下面的commit掉数据的流程：

  ```
  procedure commit(tsr, write-set) 
    tsc ← TM.commit(tsr, write-set) 
    if tsc != abort then //may return abort 
      if CT.check&mutate(tsr, commit, nil, tsc) = ⊥ then
        tsc ← abort
    // post-commit
    for all keys k ∈ write-set do
      if tsc = abort then 
        ds.remove(k, tsr) 
      else 
        ds.put(k, tsr, commit, tsc) 
    CT.remove(tsr) // garbage-collect CT entry
  ```

  原子的方式写入commit的记录。如果失败则需要abort。在abort的流程中，为直接清除之前写入的数据，而成功的话，将之前写入的数据状态改为comitted。

* 读取操作的时候要稍微复杂一些，从最新到老的读起。遇到一个commit列不为nil的，且这个版本的commit ts小于目前事务的start ts，则直接表明其读取到了一个合适的值。如果commit列为nil，则可能事务已经commit，也可能还没有commit，需要读取commit table判断。读取如果commit table的时候，如果其中没有对应记录，则事务可能已经abort or 还在进行中。在合适的时候这里会尝试一个强制abort的操作，需要使用CAS的方式将这个start ts对应值置为abort。如果CAS失败，则可能是前面的事务已经committed，重新读取这个状态。如果成功强制abort，则这个版本的数据被忽略。Reader尝试abort事务的时候，可能是之前的事务在这个过程中已经完成了事务commit，已经commit table数据的清理。这样就需要在设置了强制abort之后，再去读一下数据，如果数据被commit，则需要将之前写入事务的abort状态删除。

 另外一个优化设计单个key操作的优化，主要增加了下面的一些API。对于brc和br，实现就是下层使用的HBase的Get操作。区别在返回version与否，而brc就是单个key的写入，wc操作则是单个key的CAS操作。对于bwc操作，遇到了前面处于不明确状态的写入，则会直接abort，降低额外操作带来的开销，另外会和wc的操作，其会生成对应key一个新版本的数据，这个版本号/timestamp要求比之前已经commit的要大，而且要求和不走fast path的事务之间的冲突能够检查出来。Omid的做法是：将时间戳范围两个部分，一部分为global version，一部分sequencer number。bwc操作会读取对应key最新的版本，并将其增加其commits ts并写入，如果超过了sequencer number能够表示的范围，则需要abort。这个时候可以再走之前的事务路径重试。另外的问题涉及到冲突检测的。这里的处理方式是维护一个max version，不小于key已经commit的版本，put操作会增加这个max version，读操作也会增加max version到不小于读操作的timestmap，普通的写事务操作的时候，检查通过判断max version和事务的timestamp，要求max version不大于这个事务的时间戳。另外fast path的写入可能导致读不符合snapshot的语义(出现了新的可以满足snapshot要求的版本?)，使用fast path commit之后，Omid的三怕手头的语义就是有了一些变化。

```
brc(key) – begins an FP transaction, reads key within it, and commits.
bwc(key,val) – begins an FP transaction, writes val into a new version of key that exceeds all existing ones, and commits.
br(key) – begins an FP transaction and reads the latest version of key. Returns the read value along with a version ver.
wc(ver, key,val) – validates that key has not been written since the br call that returned ver, writes val into a new version of key, and commits.
```

### 0x12 一种可能的设计

 总结到这里的时候，OceanBase已经开源，但是开源之后的文档依然是很垃圾。这里从其模糊不清的文档描述和其它的一些网上能够找到的描述来估计一下其分布式事务大概的一个设计。OceanBase分布式事务实现的基本方式也是2PC的方式，这里也使用了一些优化方式使得可以在第一步prepare之后就可以返回给client，后面实际的commit可以异步完成。其可能是这样的实现思路[3]：

* Prepare之后，要求返回OK之后事务后面是可以commit的，这个2PC的语义就要求这样，正确实现的情况下就可以满足；另外一个是协调者在返回给client之后commit事务之前crash的话，如何保证事务被最终提交。这里使用的方式Percolator之类的有相通的地方，但不相同。其基本方式是在prepare的数据中记录上前事务的参与者列表。节点在收到这个prepare请求之后，需要把相关的数据，redolog以及参与者list都持久化。
* 正常的情况下，commit的时候会向所有的s参与者发送一条commit的时候，这样参与者就可以持久化这个日志。如果没有收到这个commit的信息，前面的prepare写入的数据都的保存着。因为后面在消费log的时候，如果没有发现对用commit日志，则通过之前写入的参与者list中其它的参与者询问情况，在可以commit的时候，参与者之间会协调将事务commit。这样如果一个事务没有commit信息前面的prepare就不能随便删除。如果已知所有的参与者都commit/abort这个事务之后，对应的prepare写入的数据就可以删除。
* 这里如何处理timestamp的问题呢?这里可以根据Clock-SI的方式来进行猜测：Clock SI中事务commit ts的确定是获取了参与的每个partition的最大的local ts。而这里也可以使用同样的思路，暂将每个参与者本地最大的local ts称之为prepare ts，将其和prepare的请求持久化并返回给协调者。航母commit的时候，使用收到的prepare ts中最大的一个作为commit ts。

#### 也使用Timetamp Server

 但是这里读取操作的timestamp改如何选择呢？如果只是read committed的话，直接读取每个分区最新已经committed的数据应该就可以了？有没有其它的不满足的cases？但是这样的话，snapshot isolation是肯定满足不了的，因为不同节点读取到的不一定是同一个snapshot。另外在OceanBase另外的一些描述中[4]，其也提到了使用一个中心化的timestamp server，Global Timestamp Service，称之为GTS[4]。这里如何和前面提到的2PC方式结合的网上找不到相关的资料，单可以根据前面TiDB Async Commit的思路猜测一下。因为工程的，在实现思路上往往选择的是经过了验证了一些方法，种类未必会很多。这里GTS可以:

* 同样的也在事务开始的时候向GTS获取一个时间戳，暂时和前面TiDB一样称之为prewrite ts。后面进入2PC的papare阶段，协调者这一步的操作和前面的基本相同。参与者选择prewrite ts和本地local ts中最大的一个作为前面的prepare ts，向用户返回并持久化。参与者另外改变的一点是，如果local ts比prewrite ts要小的话，需要将local ts增大的不小于prewrite ts。而读取的话，就可以选择从GTS获取一个start ts/read ts来读取一个snapshot。
* 在描述不清晰的文档[4]中还谈到了对GTS的两种优化方式，一种方式是一个节点上会维护一个Global Committed Version，如果一个操作值涉及到一个节点时候，就可以直接使用这个Version作为read ts。这里实际就是在只有一个节点参与的情况下，直接读取最新的版本就可以满足snapshot isolation。另外一个优化是获取timestamp的时候可以批量处理，并且和提前获取，平摊/隐藏请求GTS带来的延迟。

这样从一种没有TSO/GTS方式出发，和从前面一开始有TSO/GTS的方式，两者改进到了一种很类似的方式，这样的方式本质是这两种方式的结合。

## 参考

1.  https://github.com/tikv/sig-transaction/tree/master/design/async-commit，Doc: TiDB Async Commit.
2. https://book.tidb.io/session1/chapter6/big-txn-in-4.0.html. Doc: 大事务.
3. https://github.com/tikv/sig-transaction/blob/master/design/async-commit/parallel-commit-known-issues-and-solutions.md, Doc: 问题.
4. Taking Omid to the Clouds: Fast, Scalable Transactions for Real-Time Cloud Analytics, VLDB '18.
5. http://oceanbase.org.cn/?p=195, Blog: 两阶段提交的工程实践.
6. https://open.oceanbase.com/docs/community/oceanbase-database/V3.1.0/global-timestamp-service, Doc: 全局时间戳服务.

