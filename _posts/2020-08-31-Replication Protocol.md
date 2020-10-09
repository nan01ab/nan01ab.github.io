---
layout: page
title:  Replication Protocols
tags: [Distributed]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## Hermes: a Fast, Fault-Tolerant and Linearizable Replication Protocol

### 0x00 基本思路

 这篇Paper提出的Linearizable Replication Protocol比较有意思，和经典的Paxos、Raft之类的方法比较起来，Hermes看起来要简单不少，而且还可以支持从任意一个节点写入操作，而不用和Paxos、Raft一样制定一个Master/Leader。与Paxos、Raft之类的通用的共识协议不同，Hermes更像是一个为Key-Value Store设计的 Replication Protocol。Hermes协议中的一些基本操作如下图，描述如下：

* 总体的角色分为coordinator和follower两种。客户端请求一个写入操作的机会，向coordinator发动一个写入请求，coordinator将这个写入请求作为一个 Invalidation (INV)消息广播大其它的节点，这些节点称之为follwers。Cordinator本地写入这个Key会处于Write状态。INV消息接受到的时候，这个数据处于无效的状态。Coordinator在INV消息发出之后，等待follwers的ACKs。在收到所有的ACK之后，广播一个 Validation (VAL)消息给其它节点，完成写入操作。如果从这两步的操作来看，这个就是一个2PC的操作。
* 由于Hermes协议支持从任意节点写入，所有需要一种机制来处理其写入冲突的方式。这里使用的是常见的logical timestamp。这里的每个写入操作的消息都会带有一个递增的logical timestamp，使用lamport lock来实现。这个timestamp逻辑上分为两个部分，[v, cid]。v为这个key的version number，每次操作递增，另外一个cid为coordinator的node id。两个及以上的节点对一个key进行写入操作的时候，这样cid更大节点的写入会成功。
* 在没有冲突的情况下，Hermes的写入操作话费1.5个RTT，即INV→ACK→VAL的流程。这样这里只要接受到所有的ACK，写入就会被视作完成了。这样的效果是在一些节点故障or网络分区的情况下，可能造成follwers对应key的状态一直处于INV状态。为了处理这个问题，Hermes使用了两种机制，一种是对于一个写入操作来说，如果遇到了之前的INV值，后面的写入会带上新的值，follwers接受到消息之后还是会处于Invalid状态。另外的是logical timestamp使得对于一个Key的写入操作保持了一个全局的顺序。另外如果一个节点发现一个Key处于INV状态太长的时间，它可以作为coordinator的角色，带上原来的original timestamp，replay这个write。
* 对于follwer接受到写入请求来说，它会先比较这个操作中的logical timestamp，只有在logical timestamp更大的情况下，才会接受这个请求，并将状态置为Invalid，而如果之前处理Write状态 or Replay状态，要置为Trans状态。无论follwer本地进行上面操作，都会回复一个带上INV消息的logical timestamp，也就是如果是这些写入操作接受的话，就是这个请求中带的，如果没有接受的话，会带上之前的。Follower接受到一个VAL消息的时候，如果时间戳对得上，则将Key置为Valid状态，其它情况下这个VAL消息会被忽略。
* 对于读取操作来说，它可以读取任意一个节点，如果这个Key处于Invalid状态，会一致等待，为Valid状态的时候会返回请求。

![](/assets/png/hermes-write.png)

### 0x01 故障处理以及成员变更

  故障处理方面，主要总结了这几个方面的处理：1. 网络方面的原因，比如丢失和乱序等。Coordinator对于一个请求会维护了一个超时时间，对于超时没有回复的INV请求，会重试操作。而对于VAL请求，则有Replay机制来处理，这个VAL请求的超时时间由folloer来计时。2. 对于网络分区的情况，前面coordinator处理写入请求的时候，是要等待所有的followers确认INV消息，这种格式简化了一些异常情况的处理，但是却不利于处理网络分区的情况。对于这样的复制协议来说，是很难去区分网络分区还是节点故障的，所有处理办法就是将其当作是一种情况来处理，即一个/一些节点不能正常服务了的情况。

* 对于网络分区 or 节点故障的处理，这里和要求全部ACK INV消息的特点有些冲突。Hermes的处理方式是使用Membership Lease的方式。意味着这两种故障的情况下，这个Lease的时间段内，不能正常提供服务了，

  ```
  ... the membership can only be reliably updated in the primary partition – due to its majority-based protocol – and does so only after the expiration of the membership leases. Thus, replicas in a minority partition stop serving requests before the membership is updated and new requests are able to complete only in the primary partition.
  ```

  这里也就将网络分区、节点故障以及成员变更都归一到同一个成员变更的操作。

* 成员变更操作，Hermes处理成员变更操作的方式称之为m-update，即使用一个majority-based protocol的协议。这个的操作带有一个lease renewal, 一个 live nodes的list 以及一个递增的epoch_id。在接受带这个信息的节点就可以作为coordinator服务。有些请求可能跨越不同的Membership Lease，这种情况下，read Valid状态的Key，则处理方式不变。而write or read Invalid状态的Key，需要等待live nodes的节点手到 m-update的消息。

  ```
  In contrast, writes or reads that require a replay (i.e., targeted key is Invalid) are effectively stalled until all live nodes as indicated by the membership variable receive the m-update. This is because writes and write replays do not commit until all live replicas become operational and acknowledge their INV messages.
  ```

* 在不同的membership lease过渡的阶段(transition period)，如果一个节点没有接受到m-update请求，会导致这个节点拒绝INV消息。因为这些消息的epoch_id会比本地的更大。在实际的成员变更中，Hermes还引入shadow replica这样的机制来保证平稳过渡。

在基本的操作之外，Hermes还提供了对Read-Modify-Writes操作的支持。这样的操作会在消息中带上一个RMW flag。read-modify-write (RMW)操作一般是为了实现对于一个Key的CAS操作，有不少的用途。RMW操作在Hermes中类似于Write操作，不过在遇到写入冲突的时候，可能会abort。对于RMW操作，Hermes只会提交并发更新中有highest timestamp的请求。为了实现RMW的语义，这里要满足(1) writes always commit, and (2) at most one of possible concurrent RMWs to a key commits这两点。在操作的时候做了一些改动：

* Metadata，消息的Metada中添加了一个RMW flag；
* C-ts，coordinator广播一个请求的时候，对于logical timestamp，write请求会增加2，而RMW请求增加1；
* F-rmw-ack，follwer在接受到这个请求的时候，本地logical timestamp等于小于消息中的logical timestamp时候，才回复一个INV的ACK。否则回复一个根据本地状态的生产的ACK (i.e., same message used for write replay)。
* C-rmw-abort，coordinator在folloers回复的ACK里面发现更高的logical timestamp的时候会abort这个操作；
* C-rmw-replay，RM reconfiguration之后，coorinator需要replay pending中的RMW操作，之前的ACK会丢弃。

![](/assets/png/hermes-message.png)

Paper中对于Member Ship，利用了其它协议的一些方法，

```
  Membership-based protocols are supported by a reliable membership (RM), typically based on Vertical Paxos. Vertical Paxos uses a majority-based protocol to reliably maintain a stable membership of live nodes (i.e., as in virtual synchrony ), which is guarded by leases. 
```

### 0x02 评估

 这里的信息可以参看[1].

## 参考

1. Hermes: a Fast, Fault-Tolerant and Linearizable Replication Protocol, ASPLOS '20.

