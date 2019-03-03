---
layout: page
title: Viewstamped Replication
tags: [Distributed]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## Viewstamped Replication

 Viewstamped Replication(VR)是一个适用于故障-停止的异步系统中的一个关于复制的算法，发布于80年代[2]。
 在论文[1]中有一段这样的话:

```
  VR was originally developed in the 1980s, at about the same time as Paxos [5, 6], but without knowledge of that work. It differs from Paxos in that it is a replication protocol rather than a consensus protocol: it uses consensus, with a protocol very similar to Paxos, as part of supporting a replicated state machine. Another point is that unlike Paxos, VR did not require disk I/O during the consensus protocol used to execute state machine operations. 
```

特别是其中的：

```
it is a replication protocol rather than a consensus protocol
```

 所以网上一些将其和Paxos之类的consensus protocol对比，有时候是把它当作了consensus protocol，这是不正确的。



### Background & Overview 

  VR中一组保存副本的节点称为Replica Groups，在一个数量为2f + 1的replica groups，VR最多可容忍f个副本出错。VR中每一个副本必须开始于一个初始的状态，而且操作是确定性的，如果可以满足副本上执行操作的顺序相同，则各个副本就处于同样的状态。对于一个复制协议来说，难点在与在各个副本上以相同的顺序执行操作。
  VR使用的方法是选择一个primary，用primary来处理client的请求，决定处理的顺序，而对于其它的副本，则仅仅是接受primary的处理好的顺序。正常的情况下这样没有问题，那么primary故障的情况下怎么处理呢? VR解决这个问题的方法是引入了一个view的概念。VR中每个view有且仅有一个固定的primary，如果药更换peimary，则通过执行view change，可以使系统进入下一个view，并选出新的primary取代旧rimary。VR的副本会根据一定的规则判断当前的primary是否正常工作，在判断为异常时，就是进入一个view shange的过程用于更换故障的peimary。为了保证跨view的情况下，系统也能正常工作，在之前view中已经执行的操作必须正确的反映在新的view之中，VR使用的方法是新primary必须等待直到至少f + 1个副本(包括其自身)已经执行初始化了新view的情况下才能执行操作。对于无法恢复并继续处理的节点，VR也提供了一种方法。
  总的来说，VR使用了3个子协议来处理各种情况:

        1. 正常情况下的的处理流程；
        2. view change以选择新的primary；
        3. 恢复协议。



### The VR Protocol 

 先来看看VR中的一些定义:

1. The configuration，包含2f + 1个副本IP排序之后的数组;
2. The replica number，副本IP地址在配置中的索引；

3. The current view-number，view-number，初始化为0；
4. The current status, 当前状态，normal, view-change, or recovering之一； 
5. The op-number assigned to the most recently received request, 最近收到请求的操作好，初始化0；
6. The log，op-number条目的数组，包含到目前为止按指定顺序收到的请求；
7. The commit-number，最近提交操作的op-number；
8. The client-table，记录每个客户端其最近的请求的数量，如果请求已被执行，则为该请求发送的结果。



### 正常执行流程

  当primary正常工作时，VR稳定运行，副本之间通过包含了自己当前所处的view-number的消息通信，(1)仅当收到的消息中包含和自己所知吻合的view-number时副本才会处理该消息;(2)如果收到来自旧view的消息，副本简单丢弃该消息;(3)如果收到更新的view的消息，则该副本落后了，这时需要执行state transfer过程来赶上系统的最新状态。
  正常运行中，VR执行的流程(这里基本就是论文[1] 4.1节的一些内容)

1. client向primary发送⟨REQUEST op, c, s⟩，其中，op: 需要运行的操作，c: client-id，s: 分配给请求的request-number;

        2. primary接收到消息后，对比request-number和本地记录中该client最近的一次请求，如果新请求不比本地记录新，则拒绝执行该请求，并将之前请求应答返回给client；
        3. 如果新请求的request-number满足条件，primary为请求确定op-number，然后将该请求添加到本地log中，并用它来更新本地记录中该client的最新请求。然后，primary向所有backup(非primary的副本)发送消息⟨PREPARE v, m, n, k⟩，v: 当前的view-number，m: client发出的请求消息，n: op-number，k: commit-number，代表最近提交op-number。
        4. backup按顺序处理受到的PREPARE消息: 只有在更早比op-number n更早的都处理了之后，才能处理此消息，然后递增本地的op-number，将请求添加到本地log的末尾，更新本地对该client的请求记录，最后向primary恢复回复⟨PREPAREOK v,n,i⟩消息以指示这个操作和所有早操作都在本地准备完成了。
        5. primary在收到*超过f个*(最少f+1)PREPAREOK消息后，然后在执行所有较早的操作之后，执行client提交的操作，并递增commit-number，最后向client返回应答⟨REPLY v,s,x⟩，其中v: 当前的view-number，s:分配给请求的request-number，x: 操作的执行结果，primary会将该结果保存在本地用于处理重复请求。
        6. 一般情况下primary通过发送下一个PREPARE消息来通知backups确认提交，当primary长时间没有收到client的请求时，primary就会向backup发送一个⟨COMMIT v, k⟩信息来通知确认提交。v: view-number,k: commit-number 注意这里的commit-number = op-number。
        7. 当一个backup收到确认提交的信息之后，如果本地日志中有该请求的信息，则执行commit，然后递增commit-number，再更新本地client请求的结果。如果本地日志中没有，则会等到执行完更早的操作(这种情况下时落后的，有更早的操作没有完成)，backup不会回复client。

```
                             |               |              |
                    Request  |    Prepare    |  PrepareOK   |   Reply
                             |               |              |
Client         +----------------------------------------------------------
               |             |               |              |
               +-----------> |               |              +------------>
                             |               |              ||
Primary        +----------------------------------------------------------
                             ||              | |-------->   |
                             |-------------> | +----------> |
                             ||              | |            |
Replica 1      +----------------------------------------------------------
                             ||              | |            |
                             +-------------> +-+            |
                             |               |              |
Replica 2      +----------------------------------------------------------
                             |               |              |
                             |               |              |
```

.

### View Changes 

Backup根据一定的规则判断primary故障之后，就会进入view change阶段，执行的流程如下:

1. 一个backup i 认为primary故障后，会递增view-number，设计自身的状态为view-change，向其它所有的backup发送⟨STARTVIEWCHANGE v, i⟩消息，v: 新的view-number。一个副本根据自身的定时器或者收到其它副本的比自己view-number更大的STARTVIEWCHANGE 或者DOVIEWCHANGE 消息之后进入view-change状态；
2. 当副本从其他副本收到f个包含view-number的STARTVIEWCHANGE消息时，将向新视图中的主节点发⟨DOVIEWCHANGEv，l，v'，n，k，i⟩消息，v: view-number，l: 日志，v': 状态正常的最新视图的view-number，n: op-number，k: commit-number；
3. 当新的primary接收到f + 1个DOVIEWCHANGE消息(包括自身)时，它将其view-number设置为消息中的view-number，并选择具有最大v'的消息中l的作为新的log，如果多个消息具有相同的v'，则选择其中n最大的一个。同时，将其op-number设置为新log中最新条目的op-number，然后将自身状态修改回normal，并向其他副本发送⟨STARTVIEW v,l,n,k⟩消息以通知其它副本view change完成，其中，l: 新log，n: op-number，k: commit-number。
4. 新的primary开始接受客户端请求。它还执行以前没有提交的操作，更新其client-table，并将答复client。
5. 当其他副本收到STARTVIEW消息时，将其日志替换为消息中的日志，将op-number设置为日志中最新条目的op-number，将其view-number设置为消息中的view-number，更改状态为normal，并更新client-table中的信息。如果日志中存在非提交操作，则会向主节点发送⟨PREPAREOK v，n，i⟩消息，其中，n: op-number。 然后，它们执行之前没有执行的已知被提交的所有操作，并更新其client-table中的信息。

### Recovery 

 一个副本Crash之后恢复需要重新加入，对于这种情况，VR使用了Recovery Protocol:
  1. 一个恢复中的副本i发送一条⟨RECOVERY i, x⟩信息个所有的其它副本，其中x: 一个临时值；
  2. 其它副本只能在自身的状态时normal的情况下才能回复这个消息，回复的消息是⟨RECOVERYRESPONSE v, x, l, n, k, j⟩，其中v: view-nimber, x: 之前的临时值，j：副本。此外只有当j是primary的情况下，有l: log，n: op-number, k: commit-number, 其它节点都是nil；
  3. 恢复中的副本只有在收到啦至少f+1个RECOVERYRESPONSE消息之后且这些消息都包含了之前的x，然后使用primary的消息更新自身，然后设置自身状态为normal，完成recovery。

 这个恢复协议的成本很高，因为log可能非常大，论文中说明了一种优化的方法，具体参考论文[1]。

### Reconfiguration 

  对于可以恢复的副本，可以使用Recovery的协议，然而有些Crash是不能恢复的。[1]中说明了重配置协议，允许Reolica Groups中的成员随着时间的推移而改变。比如用来处理这些情况: (1)需要重新配置来替换无法恢复的故障节点，(2)如使用更强大的计算机作为副本，(3)或将副本定位在不同的数据中心,(4)重新配置协议也可用于更改阈值f。
  重新配置由特殊客户端请求触发，该请求通过旧组的正常情况协议运行。当请求提交时，系统转移到一个新的epoch，处理客户端请求的任务转移到新组。新组不能马上处理客户端请求，直到新的副本必须知道了在上一个时期提交的所有操作，二旧的副本不能马上关闭，知道状态转移完成
  要处理重新配置，需要将一些信息添加到副本状态中:
  ```
  1. The epoch-number, 初始化0；
  2. The old-configuration, 初始化空。 
  ```
  另外还有增加了另一个status: transitioning 。副本将在下一个epoch开始的时候将其状态设置为transitioning。新的副本在新的epoch开始时使用old-configuration进行状态转移。这样一来，新节点就知道在哪里得到状态信息。当新epoch中replica group的副本的log都更新到这个epoch开始时，副本就将其状态设置为normal。
现在，每条消息都包含一个epoch-number 。副本只处理与它们所在的epoch-number匹配的消息(来自client或其他副本)。如果接收到具有epoch-number的消息，则需要转移到到该时期，如果收到具有较早epoch-number的消息，则它们丢弃该消息，同时会通知发送者有更新的epoch-number。


​    重新配置的请求由特殊客户端c 发送，例如管理员的节点，该节点向当前主节点发送⟨RECONFIGURATION e，c，s，new-config⟩消息。其中，e: c已知的当前时代，s: c的请求号，new-config: 新组的所有成员的IP地址。Primary只有当s足够大(基于client-table)，e是当前epoch-number时才接受该请求。此外，如果new-config少于3个IP地址，主节点将丢弃该请求。

   如果primary接受了该请求，它会按通常的方式处理client请求的方式处理这个请求，但是有两个区别：首先，(1)主要立即停止接受其他客户端请求，重新配置请求是当前时代处理的最后一个请求。其次，(2)执行请求不能导致an up-call to the service code，重新配置仅影响VR状态。
具体的Reconfiguration Protocol可以参考论文[1]，到这里这算法已经介绍的差不多了，更加具体的信息还是看原论文. 总的来说，论文[1]以步骤的形式给出了VR的操作，描述还是很清晰的。



## 参考

1. Viewstamped Replication Revisited: http://www.pmg.csail.mit.edu/papers/vr-revisited.pdf
2. Viewstamped Replication: A New Primary Copy Method to Support Highly-Available Distributed Systems.

