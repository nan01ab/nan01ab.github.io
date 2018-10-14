---
layout: page
title: A Quorum-Based Commit Protocol
tags: [Distributed]
excerpt_separator: <!--more-->
typora-root-url: ../
---



# A Quorum-Based Commit Protocol
  在distributed computing中，quorum是指分布式执行时必须获得的最小的票数[2]，常被用于分布式的commit protocol和replica control。这里我们来看看论文[1]描述的一种quorum-based commit protocol。

>

## 介绍

  这个Quorum-Based Commit Protocol(下文简称protocol or 协议)使用quorum的方式来解决错误发生之后的冲突。当有一个错误发生之后，一个事务只能在被一个quorum commit的情况下commit，这个quorum称为commit quorum，表示为V(C)，同理，一个事务abort只有在一个quorum abort的情况下abort，这个quorum称为abort quorum，表示为V(A)。
  这个protocol有以下的一些的特点：
    1. 这是一个集中式(centralized)的协议(也就是说存在中心节点)；
    2. 当所有的错误(failure)被解决之后，协议最终会终止；
    3. 这是一个阻塞的协议，只有当错误被修复之后才能继续运行。

  此外，协议可以从多种的失败中快速恢复，但是主要关注的错误的网络分区，主要是以下两种类型: 节点失败和消息丢失，这个两类情况都可以被视为是网络分区错误。



## 协议

  在介绍这个协议之前，先来看一看经典的2PC协议[3]，2PC可以用下图简单地表示:
```
                 +---------+
                 |         |
                 |   q     |
                 |         |
                ++---------+
                |          |
+----------+    |          |   +----------+
|          |    |          +-> |          |
|   w      | <--+              |   a      |
|          +-----------------> |          |
+------+---+                   +----------+
       |
       |
       v
+----------+
|          |
|   c      |
|          |
+----------+
```
  2PC协议包含了以下的几个状态,初始状态(q)、等待状态(w)、abort状态(a)和commit状态(c)。一个事务中，可以分为可提交(committable)和不可提交(noncommitable)两种状态，2PC中，相同只有在所有节点处理可以提交的状态时整个系统才处理可提交的状态，也就是说，2PC中只有c状态时一个可提交状态。



### 提交

  现在来看看这个quorum-based的协议，前面提到，这个协议中存在V(C)和V(A)。不同于2PC中的操作是对于所有节点而言的，这个协议中的操作时对于V(C) or V(A) 而言的。此外，这个协议还有以下的要求:
    1. V(A) + V(C) > V， 0 < V(C),V(A) <= V, V表示总票数；
    2. 任意一个节点处于commit状态时，至少有一个处于可提交状态的quorum；
    3. 任意一个节点处于abort状态时，至少有一个处于不可提交状态的quorum。

这些要求保证了协议会在一个一致的状态下结束(all commit or all abort)，其中，第2条可以理解为有2条子要求组成:
  2.1 在第一个节点commit之前，必须有一个到达可提交状态的quorum；
  2.2 在任意节点提交之后，必须维持这个quorum。这一条表明了一个节点可以安全的有可提交状态变为不可提交状态只能在没有任何节点已经commit的情况下。
  同理，对于第3条要求，可以类比到第2条。
  我们可以发现，2PC协议提交操作时不能满足上面提到的要求(不能保证所有节点最终都是提交了or最终都是没有提交)。为了解决这个问题，在这个协议中添加了一个prepared to commit（pc)的状态，所以，这个协议用图表示如下:
```
                 +---------+
                 |         |
                 |   q     |
                 |         |
                ++---------+
                |          |
+----------+    |          |   +----------+
|          |    |          +-> |          |
|   w      | <--+              |   a      |
|          +-----------------> |          |
+------+---+                   +----------+
       |
       |
       v
+----------+
|          |
|  p       |
|          |
+------+---+
       |
       v
+----------+
|          |
|  c       |
|          |
+----------+
```
  添加的p状态类似于3PC[4]中的对应状态，到了这一状态表了节点时可以执行提交的。此外，这个协议是一个悲观的协议，如果任意节点在w状态时失败，事务会被abort。



### 恢复

协议中的恢复包含了2个方面，包括错误发生之后的termination protocol和错误恢复之后的merge protocol。



#### Termination Protocol

  网络分区发生之后(论文中只讨论了网络分区的错误)，当一组节点发现它们在一个网络分区之中之后，它们会先选举中一个surrogate coordinater，至于具体如何选举，则不是这个协议所关心的。
  当surrogate选举出来之后，它要做的是继续执行。Termination protocol主要因为两个原因变得更加复杂，第1个原因是一个surrogate不知道全局的情况，也不知道一个事务是否可以提交，第2个原因是会有多个surrogate在各自分区中操作。  
  对于第1个问题，这个分区只有在这个分区包含了处于可提交状态的节点才可以尝试提交。对于第2个问题，surrogate必须明确地组建abort quorum，一个节点通过进入prepared to abort状态来表明自己的意愿（原文: A surrogate must explicitly form abort quorum. A site indicates its willingness to participate in an abort quorum by moving into a prepared to abort state）。
  surrogate通过轮询每一个节点来获取它们的状态信息，如果有任意节点已经commit(or abort)，事务也应该立即在所有节点上commit(or abort)。如果没有节点已经commit or abort，则当至少一个节点处于prepared to commit状态，且prepared to commit状态 + wait状态的节点大于等于了V(C)，surrogate会尝试将所有的节点的状态转为prepared to commit状态，在没有其它错误的情况下，它会提交这个事务，否则就阻塞(等待错误修复)。同理，对于abort也是这样。
  在增加了这些之后，协议的图示如下:
```
            +----------+                   +----------+
            |          |                   |          |
            |   w      +---------------->  |  pa      |
+-----------+          |                   |          |
|           |          +---------+         |          |
|           +------+---+         |         +-----+----+
|                  |             |        +      |
|                  |             |        |      |
|                  v       +--------------+      v
|                          |     |
|           +----------+   |     |        +-----------+
|           |          |   |     +----->  |           |
|           |  pc      |   |              |           |
|           |          | +------------->  |  a        |
|           |          |   |              |           |
|           +------+---+   |              +-----------+
|                  |       |
|                  v       |
|           +----------+   |
|           |          |   |
+---------> |  c       |   |
            |          | <-+
            |          |
            +----------+

```



#### Merge Protocol

 恢复之后的merge protocol比较简单，在发现网络分区问题被修复之后，执行上面描述过的termination protocol。新的coordinator可以直接从旧的coordinator选选取，比如选取最少节点分区中的coordinator。



## 总结

  Quorum的思想在分布式中一个比较基础的思想，也有很多实际的应用，比如在Amazon Dynamo。不过在这个协议之中，使用quorum并没有完全解决分布式系统中提交存在的问题，比如节点可能一致阻塞在termination protocol中，依赖于错误的修复来解决这个问题，所以论文中也只是说这个是一种 "可以快速恢复" 的算法。



## 参考
1. A Quorum-based Commit Protocol: https://ecommons.cornell.edu/bitstream/handle/1813/6323/82-483.pdf?sequence=1
2. Quorum: https://en.wikipedia.org/wiki/Quorum_%28distributed_computing%29
3. 2PC: https://en.wikipedia.org/wiki/Two-phase_commit_protocol
4. 3PC: https://en.wikipedia.org/wiki/Three-phase_commit_protocol
5. Amazon Dynamo: https://en.wikipedia.org/wiki/Dynamo_(storage_system)

