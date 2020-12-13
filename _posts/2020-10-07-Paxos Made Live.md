---
layout: page
title: Paxos Made Live - An Engineering Perspective
tags: [Consensus, Distribution]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## Paxos Made Live - An Engineering Perspective

### 0x00 Algorithmic Challenges

 这篇Paper是比较有名的一篇，应该是早就读过了。Paxos Made Live给出的是Multi-Paxos工程实践上面的一些总结。Multi-Paxos在是Paxos上面改进而来的用来在多个value上面达成一些的算法。Multi-Paxos可以设计为在一段比较长的时间内指定一个coordinator，并尝试让这个结点的coordinator角色不改变。这里将这种的coordinator称之为Master。使用了这个优化之后，也就是加入了一个Master的角色，一般情况下一个Paxos instance只会要求一次磁盘写入。主要的设计下面总结了一些Algorithm上面需要处理的问题：

* Master leases，使用basic Paxos来实现数据复制的时候，读取数据也要求执行一个Paxos instance。这样来决定读取能过读到最新的数据。如果没有其它的改动，直接读取Master上的数据副本是不行的，因为其它那些的结点可能已经选举出了另外的Master，而且没有通知到这个Master。为了处理这个问题，这里使用了master leases的机制。在这个master leases机制的限制下面， 只有有这个lease的结点才能submit value，其它的结点submit则会被拒绝。这样就能保证Master一定包含了最新的数据。在一个lease要过期的时候，Master会尝试去续租，一般通过heartbeat来处理。Paper中描述的一般情况下，一个Master的lease租约可以续约上几天的时间(不是一次lease的时间)。在这里的实现中，一个副本会隐式地给上一个Paxos instance的master一个lease，这个lease有效的时候会拒绝来自其它结点的请求。Master上这个lease的有效期会短于其它结点上面通过lease的有效期，这个是为了处理clock drift的问题。

* 这里会有一个可能的问题导致系统的不稳定，一个Master暂时的和其它结点断开连接的时候，其它结点选举出来一个新的Master，这个master会使用一个sequence number。如果这个之前的Master有和其它的结点连接上了，使用一个更大的sequence number，使得其它的Master失效。这样可能反复，导致系统的不稳定。这种情况下，在网络不佳的情况下容易变成经常快速地Master变更。这里使用的优化方式是Master会周期性的通过一轮Paxos增大自己的sequence number。这种在其它的Consensus算法实现中也是一个需要优化的问题，比如在Raft中PreVote优化，避免一些不必要的Leader切换。

* 这里还谈到了非Master副本的lease，这篇Paper中提到这个还没有实现。这种思路在后面被另外的研究发表了"Paxos Quorum Lease"的文章，其基本思路也是获取一些keys本地读的权限。

  ```
  Note that it is possible to extend the concept of leases to all replicas. This will allow any replica with a lease to serve read requests from its local data structure. This extended lease mechanism is useful when read traffic significantly exceeds write traffic. We have examined algorithms for replica leases, but have not implemented them yet.
  ```

* 当一个请求提交到这个系统中的时候，Master接受到这个请求，然后处理这个请求到最后的更新数据库。这个过程中这个Master可能已经不是Master了，并且可能是失去了Master的身份然后又变回了Master(ABA?)。这种有Master切换的情况下，需要abort掉in-coming的请求，or 重新处理这些请求。为了处理这个问题，这里使用了Epoch的方式。系统会维护一个global epoch number。Master在一个持续的时间内为同一个epoch的Master，这个期间处理的请求才会有同样的epoch number。这个epoch会保存在数据库中。

* Group membership，Paper中这里只是提出了一下，组成员管理可以利用Paxos算法来实现，也有一些相关的研究。但是加入了各种异常情况之后，这个实现并不是很简单。但是这里Paper中没有详细说明。

* 磁盘故障。数据都是持久化保存在磁盘上面的，但是磁盘也不是可靠的。磁盘的故障一般分为两种类型，一种是文件变得不可访问，一种是文件内容损坏了。对于内容损坏，和一般的存储系统一样，这里会使用checksum来检查这种情况。如果是类似磁盘故障的情况，则文件就无法访问了。这个时候˙这个副本会使用新的磁盘，会以一种不参加投票的角色参与进来，进行数据的追赶操作。这种状态会持续到这个副本有了从其开始处理这个问题之后的所有Paxos instances。这样就不会发生违背其之前promise的情况。

* Group membership，在Chubby中，数据库以Key-Value的形式来保存数据，要求这个KV支持insert, delete, lookup，CAS以及scan的操作。这里从支持CAS这样的操作处理出发，对其进行拓展以支持transaction-style的操作。这里引入的一个操作原语是MultiOp。出来scan操作，其它的都是一个MultiOp的调用实现。MultiOp应用都是源自的，主要有三个部分：1. 一些称之为guard的tests，每个guard对数据库中的单个key进行测试，比如测试一个key存在与否、或者是与一个值进行比较。对于MultiOp的所有的测试都执行且返回都true的时候，才会去执行t-op，否则执行f-op；2. t-op即一些数据库的操作，每个操作可以是insert、delete、lookup这样的单点的操作；3. f-op同样是一些数据库的操作，在guard测试出现false的时候执行。

Paper这节中还详细讨论了的一个情况是Snapshot。这个snapshot的实现是一个数据库的一个snapshot，加上一个将会应用到这个数据库的一些log组成。这个log就是Paxos的log。对于已经apply到数据的log，就可以进行截断的操作了。在Chubby这样的系统中，Paxos的log最终会被apply到其它的存储结构中(Key-Value Store)，但是Paxos framework自己是不知到这个存储结构是怎么样的。这样需要来应用这个Paxos的来通知其做snapshot。在得到这个通知之后，framework会删除这个snapshot之前已经应用了的log。在crash之后的恢复操作中，通过最后面的一个存储结构的snapshot，加上重新应用截断之后的log来恢复状态。这里的。snapshot在每个结点上单独进行的。这样的实现描述起来比较简单，但是由于持久化变成了两个部分，会给实际的实现带来不少的麻烦。log是在Paxos framework的控制之下的，而存储结构不是：

* 这个的实现需要存储结构的snapshot和log保存相互的一个一致，即每个snapshot需要记录其对应到那些日志。为了处理这个问题在framework，使用了一个snapshot handle。这个handle保存了一个snapshot所有的Paxos-specific的信息。在创建一个snapshot的时候，应用需要将这个handle保存下来。在恢复的时候需要应用向framework提供这个handle。Handle实际上就是一个Paxos自己state的一个snapshot，包含了对于的instance number和membership。
* 在这个framework中，做一次snapshot主要分为三步：1. 一个应用发送一个take a snapshot的操作，它会先请求一个handle；2. 然后client对自己做一次snapshot。这个操作可能阻塞应用，也可以使用另外的线程处理这个工作，Paxos的工作也可以继续进行。这个snapshot必须是应用获取到handle时候对于的log position的系统的状态。如果在这个过程中Paxos的工作就继续进行，这个要保证应用在做快照的时候其存储结构还是可以继续更新的；3. 应用的快照处理完成之后，应用需要通知framework，之前获取的handle会作为一个参数。framework会根据这些信息截断日志；
* 为了处理snapshot的过程中失败，framework只会在其接受到最后一步的通知的时候才回去截断日志。在此之前log是一直在的，应用可以利用这个来处理一些数据完整性校验的问题。应用的snapshot的情况下，不会去通知framework截断日志。而且这种方式可以同时进行多个snapshot。
* 一个故障 or 新加入的结点会有一个catch up的操作。在catch up的时候，这个副本需要拉去缺失的log。但是这个可能已经被截断了，这个时候就需要从其它的副本来获取一个snapshot。同样地前面的snapshot handle会被使用，以获取catch up到哪里的信息。在获取了一个snapshot之后，剩余的log从leading replica获取。这个过程中leading replica有可能创建了一个新的snapshot，这样可以导致这个catch up中的副本无法获取到完整的剩余的log，就需要重新获取最新的snapshot一次。另外leading replica可能在发送snapshot的过程中可能故障了，catch up的副本需要能处理这种情况，转而向另外的副本请求snapshot的数据。另外这个需要一种获取最佳snapshot的机制。

### 0x01 Software Engineering

除了算法上面的一些工程实践的总结之外，这里还总结了一些在软件实现方面的一些总结。Fault-tolerant的算法一般很难表述准确，如果是在一个系统中，混合上其它功能的代码就更加复杂了。不仅仅实现上面，而且debug、后面的改动也会比较麻烦。这里处理这个问题通过将核心的算法分为两个状态机。这里还实现一种state machine specification language，然后使用了一个编译器将这种 specification language编译为C++的代码，并且会自动生成debugging和测试的代码。这种方式会带来不少的好处，更加容易修改。Paper中举了一个成员管理算法修改的例子：比如开始的时候这里将成员变更的操作，对于一个结点分为三种状态：等待加入、已经加入和最终离开。一旦一个结点离开成员组，就不能再加入了。这么设计的原因是一开始认为故障的结点将会在比较长的一段时间内都不能重新工作。但是实际上临时的故障很多，后面就改为在 or 不在两种状态。利用这里的specification language比较简单的实现这样的改动。初次之外，这了还使用了另外的一些方法来保证工程上的算法的正确实现：

* 运行时候一致性检查，Paper中介绍使用了一种计算database log checksum的方法。Master会周期性地计算一个checksum，然后发送到其它的结点上，其它的结点计算本地的一个checksums。由于Paxos使得操作是有序的，理论上应该会得出一个一样的checksum，有这个就可以检查不同副本之间的数据一致性。Paper中举例了三个不一致的例子，比如有人操作导致的问题，硬件故障导致的问题，or 内存访问错误导致数据错误的情况等。

* 目前来说证明一个系统是正确的是不太可能的(形式化验证还是应用的范围不大)，这里系统的问题发现主要还是依赖于测试。测试引入了两种模式：一种是safty mode，这种模式下面用于测试一致性，可以有操作错误 or 系统不可用。另外一种是liveness mode，这种情况要求系统向前运行，所有操作应该是正常完成的，系统也应该是一致的。在safty mode下测试的时候，会通过注入随机的故障来测试系统，然后停止一段时间让系统恢复。然后切换到liveness mode，这里liveness mode的测试用于测试在一系列故障之后系统不会deadlock。另外一个测试验证其fault-tolerant log，在随机数量副本的系统中，在经过一系列的网络故障、消息延迟、超时、进程crash又恢复已经文件损坏等故障的之后能过恢复。为了能重现测试，这里将一个生成的随机序列来测试，而且使用单线程来避免使用多线程带来的一些不确定性。这种测试发现了系统中的很多问题。在Chubby这样的系统中，还通过注入错误的方式模式底层系统和硬件的故障。

* 实际的系统是多线程的，前面的测试发现使得和实际的系统有差别。Paper中认为这里还有一些问题，

  ```
  As the project progressed, we had to make several subsystems more concurrent than we had intended and sacrifice repeatability. Chubby is multi-threaded at its core, thus we cannot run repeatable tests against the complete system. ...
  In summary, we believe that we set ourselves the right goals for repeatability of executions by constraining concurrency. Unfortunately, as the product needs grew we were unable to adhere to these goals.
  ```

* 容错的系统还带来了另外的测试问题，Paper中举的一个配置错误的例子，五个结点中四个是正常的，另外一个一直在catch up。系统四个结点的时候就可以正常运行，导致这种问题没有被立即发现。即系统自身的容错能力也会掩盖一些问题。

除了这些工程实现上的总结之外，paper中还提到了使用过程中的一些unexpected failures，意料之外的故障。比如在升级了一个Chubby的版本之后，期望其处理能力能够提高。但实际应用的时候负载比较高的时候，发生了一些线程饥饿导致系统频繁超时。这个有导致了频繁的master切换。这样有导致了client的请求转移到新的master，有导致了新master load太高。从而导致了级联的问题。然后再会滚到旧版本的时候，有发生了操作失误导致了丢失了一些数据。然后后面准备更新的时候，又操作失误导致用错了snapshot，又导致了丢数据。Paper中这部分总结了不少操作问题导致的故障。另外还提到了Chubby和数据库一个期望不一样的情况：如果一个Chubby结点提交一个操作到database之后，但是database这个时候失去了master的状态，Chubby希望这个操作是失败的。但是实际上数据库操作的时候，另外一个副本可能变成新的master，而操作有可能成功。这个要求处理不少integration layer的问题，和利用epoch 和MultiOp的功能，具体细节Paper中没有说明，

```
With our system, a replica ould be re-installed as master during the database operation and the operation could succeed. The fix required a substantial rework of the integration layer between Chubby and our framework (we needed to implement epoch numbers). MultiOp proved to be helpful in solving this unexpected problem – an indication that MultiOp is a powerful primitive.
```

 另外还提到了一个Linux Kernel bug导致系统故障的case。Flush操作一个小文件到磁盘的时候，如果太多buffer数据需要写到磁盘的时候，这个flush操作可能被hang住很长的时间。比如在写了一个数据库snapshot的时候。处理这种问题的方式是将大文件都拆成小的chunk来写入。

### 0x02 评估

 这里的具体内容可以参考[1].

## 参考

1. Paxos Made Live - An Engineering Perspective, PODC '07.