---
layout: page
title: PacificA -- Replication in Log-Based Distributed Storage Systems
tags: [Distributed]
excerpt_separator: <!--more-->
typora-root-url: ../
---



# PacificA: Replication in Log-Based Distributed Storage Systems
  PacificA是微软推出的一个通用的复制框架。论文[1]中强调PacificA是一个简单、实用和强一致性的。也许是基于Paxos的方法被太多的吐槽了关于复杂，难以理解难以实现，Raft、PacificA等算法都特别强调自己简单易于理解也相对容易实现。
  选择一个合适的复制算法就算选择一个正确的架构设计，简单和模块化是设计的几个要点。PacificA的设计有以下的特点:
    1. PacificA将配置管理和数据复制分离，由基于Paxos的模块来负责管理配置，主副本(primary/backup)策略来复制数据；
    2. 去中心化的错误检测和触发配置，监控流量遵循数据复制的策略；
    3. PacificA是一个通用的抽象的模型，易于验证，可以有不同的策略实现。

系统有以下的假设: 系统运行在一个分布式的环境之中，一些server组成集群，这些server随时可能失败，这里假设的情况是失败之后就停止运行(fail-stop failures)，信息在传输的过程中可能被延时任意长的时间达到，也可能丢失，可能会发生网络分区。时钟也是不可靠的。PacificA可以在一个n+1个副本的系统中最多容忍n个副本故障。

>

## 主副本数据复制

  系统将客户端的请求分为两种: queries请求只请求数据而不会更新数据，而updates请求会更新数据。所有的请求都会发送个primary，primary会处理全部的请求，而只会在updates请求的时候才会让副本参与进来。这样的主副本的策略好处就是简单易于实现。
  如果更新操作的结果是确定的，且所有的服务器都以系统的顺序处理了相同的请求集合，那么就可以实现强一致性。为了实现这个目标，primary(主副本)会赋予每一个请求一个唯一的单调递增的序号给所有的updates请求，每个次副本都的安装这个序号表示的顺序进行处理。这里可以表示为每个副本都维持了一个包含了它收到的所有请求的prepared list和一个对于这个list的一个commited point。这个list会根据序号排序，commited point之前的prepred list部分就是commited list。

>

### 正常情况情况下的查询和更新协议

  正常情况下，primary对于接受到的queries的请求，直接就查询commited list之中的结果如何回复客户端即可。
  对于updates请求，primary会先给这个请求赋予下一个的编号(编号可以初始化为0)，然后将这个请求带上编号和当前配置版本号(prepare message)发送给所有的次副本。
  在收到prepare message后，次副本会将这个请求插入prepared list(注意要按照编号的顺序)，在次副本确认这个请求被安置妥当之后，就发送给primary一个ack以告知自己已经将这个请求安置妥当。当primary在收到了*所有的*次副本的ack之后，primary就会提交这个请求(The request is committed when the primary receives acknowledgments from all replicas)。然后，primary会移动它的committed point，然后向所有的次副本发送消息通知它们已经处理完毕，在收到了primary确认成功的消息之后，次副本也就可以移动它的committed point的了。
  在这种处理方式之中，会有以下的结论：
```
  Commit Invariant: Let p be the primary and q be any replica in the current configuration, committed(q) ⊆ committed(p) ⊆ prepared(q) holds.
   即: primary已经commit的请求是已经准备好的一个子集，一个次副本已经commit的请求是primary已经commit的请求的子集。
```



## 配置管理

  全局的配置管理器复制管理所有副本组的配置，对于每一个副本组，配置管理器都会保存这个组当前的配置和配置版本。当服务器通过故障检测认为一个副本发生故障时，服务器会启用新配置；配置管理器可以将故障的副本从配置中移除；此外，服务器可以提出添加新的副本(包括过去失败的副本重新恢复)给配置管理器。这些情况下，配置管理器都会提出新的配置以及当前的配置版本好到配置管理器。只有在这个版本与配置管理中的版本号相同时，配置管理器才会处理这个请求。然后，配置管理器将会递增配置版本号，然后保存这个配置。
  在网络分区方式时，一个次副本可能尝试移除primary，而primary会尝试移除次副本。由于这个改变的决定都得有配置管理器来决定，所以配置管理只要接受第一个请求即可而拒绝之后的请求，因为在接受第一个请求之后，当前的配置版本号以及变了。
  我们可以得到以下的结论：
```
   Primary Invariant: At any time, a server p considers itself a primary only if the configuration manager has p as the primary in the current configuration it maintains. Hence, at any time, there exists at most one server that considers itself a primary for the replica group. 
  即: Primary认为自己是primary只在在配置管理中认为自己是primary的情况下，任何情况下，都只有一个的primary。
```



## 租约和错误发现

  根据上面配置管理的方法，可以知道只有配置管理器保存了准确的配置，而primary or其它任何次副本保存的消息都不一定是正确的当前配置。这种情况下，我们必须解决这样一个问题: old primary和new primary都在处理请求，这会造成错误。
  这里使用的解决方案是*leases*(租赁)，primary通过定期的向次副本发送beacons消息等待确认。Primary会在距发送beacons消息过了一段lease period之后没有收到ack情况下认为租赁过期。对于任何次副本的租赁到期之后，primary都会认为自己不在是primary，并通停止处理请求。这种情况下，primary会联系配置管理请求从当前配置中移除这个次副本。
  如果一个次副本距离上次收到primary的消息已经过了一段grace period，这个次副本会认为primary的租赁已经到期，它会向配置管理请求移除当前的primary并请求自己成为新的primary。
  假设没有时钟偏移，只要有grace period >= lease period，就可以保证primary在次副本认为已经过期之前过期。就可以解决上面的问题。
  系统的故障检测流量总是发生在需要有通信两个节点之间: 处理updates请求时主副本之间的通信，primary发送beacons消息和次副本回应ack消息时主副本之间的通信。数据处理通信业本身就可以当作beacons信息，使用只有在通信通道空闲的时候primary才需要发送beacons消息。PacificA的租约机制时一个去中心化的租约机制。



## Reconfiguration, Reconciliation, and Recovery 

 Replication Protocol的一个很大的复杂性来与就是Reconfiguration。这里，PacificA将此分为了三种情况: 移除一个次副本、移除primary和添加一个次副本。



### 移除次副本

 当primary人认为一个or多个次副本发生故障时，它会向配置管理器提出一个新的配置请求，用于移除认为可能故障的副本，在获得配置管理器的通过之后继续运行。



### 改变primary

  当一个次副本认为primary故障时，次副本会向配置管理器提出一个新配置，这个配置会把自己当作新的primary，且会把旧的primary排除在外。当这个配置获得通过时，这个新的primary只有在完成reconciliation过程之后才开始处理请求。在reconciliation阶段，新primary将提交prepared list中的没有提交的，然后会将自己的记录同步到其它副本。因为prepared的可能多了or少了，但是已经提交了的大家都是一样的，对于次副本少了的情况，需要补齐，而对于多了的情况，设sn为新primary reconciliation之后committed point指向的记录的编号，p会要求其它副本都截断prepared list到sn。根据上面的Commit Invariant，是不会发生已经提交的被截断的，只会是哪些已经prepared但是没有提交的记录。此外，Reconciliation期间也可能提出新的configuration。
  因为次副本上的prepared list总是包含了所有的已经提交的请求，通过将prepared list中的请求提交，因此reconciliation会有:
```
  Reconfiguration Invariant: If a new primary p completes reconciliation at time t, any committed list that is maintained by any replica in the replica group at any time before t is guaranteed to be a prefix of committed(p) at time t. Commit Invariant holds in the new configuration. 
```



### 添加次副本

新副本添加是要保证Commit Invariant，这要求新副本在加入之前准备好prepared list。一个简单的方法是primary暂定执行知道新副本复制完成prepared list。另外一种方法是先作为一个候选副本加入，新副本逐渐获得数据"赶上"来。



## Replication for Distributed Log-Based Storage Systems 

论文[1]还讲了Replication for Distributed Log-Based Storage Systems。有兴趣的可以具体参考论文[1]。



## Extend

### Kafka ISR
 Kafka为了保证高可用使用了ISR(In-Sync Replicas)的方法，这个方法与PacificA非常相似。具体内容可以参考Kafka问到中的Replication章节[2]。



## 参考

1. PacificA: Replication in Log-Based Distributed Storage Systems: http://www.microsoft.com/en-us/research/publication/pacifica-replication-in-log-based-distributed-storage-systems/
2. Kafka Replication Document: http://kafka.apache.org/documentation.html#replication

