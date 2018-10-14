---
layout: page
title: Distributed 0x03
tags: [Distributed]
excerpt_separator: <!--more-->
typora-root-url: ../
---



## Distributed 0x03

 很多的分布式算法中需要一个协调人来做一些特殊的决策，如何在一些进程中选举出这个协调人就是分布式选举算法的内容。



### 几个选举算法

 在这个算法中，都假定每个进程都被一个数字唯一的表示，一个进程也知道其它所有进程的标识符。



### The Bully Algorithm

  Bully算法是一个出现的比较早比较简单的算法，先通过这个算法来讨论一下分布式选举中需要解决的问题。
Bully算法假设一下条件:
1. 同步系统；

2. 进程可能随时fail；

3. 进程的fail之后停止运行，通过重启恢复运行；

4. 有failure检查，用于发现failed的进程；

5. 信息传递时可靠的；

6. 每个进程知道所有的其它进程的id和地址。

   

在Bully算法之中，有以下类别的消息[1]:
1. Election Message: 用于宣布一次选举；

2. Answer (Alive) Message: 回复Election Message；

3. Coordinator (Victory) Message: 被选举为Coordinator的进程向其它进程宣布它被选举为Coordinator。

   

这个算法的基本的过程如下:
1. 如果进程p拥有最大的id（process id)，p向其它所有进程发送Coordinator Message说明它赢得了选举，否则，p向其它的id比自己高的进程广播一个Election Message；
2. 如果p在发送了Election Message之后没有收到Answer Message，p向其它所有进程发送Coordinator Message说明它赢得了选举。
3. 如果p收到了一个Answer Message且来自一个id更大的进程，则不发送消息，等待别的进程发送来的Coordinator Message，如果一段时间之后没有收到Coordinator Message，则重复选举过程。
4. 如果p收到了一个Answer Message且来自一个id更大的进程，则回复一个Answer Message。然后p启动一个选举过程，向其它的id更大的进程发送Election Message。
5. 如果p收到了一个Coordinator Message，则将发送这个消息的进程当作Coordinator。

随着算法的运行，id较小的进程会逐渐放弃选举为Coordinator，最终，id最大的进程会赢得选举。



### A Ring Algorithm 
  这里的基于环的算法不是互斥算法中基于环的算法。在这个算法中，假定进程被排列为环，每个进程都知道另外一个进程的下一个进程(不仅仅是自己的下一个进程)，这个算法也假定消息的发送接收是可靠的。
    1. 当任意进程注意到coordinator不工作时，它发送一个带有自己process number的Election Message给它的下一个进程；
    2. 如果这个进程的下一个进程不工作，则将消息发送到下一个进程的下一个进程，如下一个进程的下一个进程，以此类推往后面的进程发送消息；
    3. 每一个接收到这个消息的进程在这个消息之中添加自己的process number，然后重复2(所有进程的process number在这个消息被表示为一个list)；
    4. 当这个消息回到最初的进程时，这个进程根据消息中process number的list，找到process number最大的进程，然后发送一个Coordinator Message重复循环过程来告知那个进程被选举为coordinator。
    5. 此外，还会重新组织环，在Election Message消息发送中被认为已经不能工作的进程会被从环中去除，也会重新添加重启之后继续工作的进程。



### 不稳定系统中的选举



### 几个例子



#### Viewstamp Replication



#### Paxos



#### Raft



#### PacificA

在PacificA中，谁是Primary是有Configuration Manager决定的，



## 参考

1. https://en.wikipedia.org/wiki/Bully_algorithm，维基百科.
2. 《Distributed Systems: Principles and Paradigms》第二版，ISBN: 0-13-239227-5.
3. https://en.wikipedia.org/wiki/Leader_election，维基百科.

