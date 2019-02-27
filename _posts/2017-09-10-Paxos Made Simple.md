---
layout: page
title: Paxos Made Simple
tags: [Distributed]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## Paxos Made Simple

  Paxos是分布式系统中的一个非常重要的算法，用于解决容错的分布式系统中对*一个值*达成一致的算法。论文[1]以一种很好的理解的方式描述了这个算法(这里讨论的是基础的paxos算法，不是其它的变种)。算法假设有以下的一些条件:

    1.  进程(or 角色)一任意的速度执行，每个都可能出错、停止运行，也可能错误之后恢复过来。所有进程可能在选定这个值值后都失败重启，它们可能在重启之后不能确定之前被选定的值，除非其中的一些进程保存下了这些信息。
    2.  信息在传输的过程中可能被延时任意长的时间达到，也可能丢失、重复，但是信息的被人不会被修改。

总的来说，这篇paper讲的还是很清晰的。

### 提出问题

  考虑在一个可能发生一些错误的一个分布式系统中，多个进程针可以提出自己的value，最终这个进程将会对这个值达成一致。当然在没有值没有被提出的时候，也就没有值会最终选定。当只有一个值被提出的时候，算法也要保证最终被选定的值就是这个唯一提出的值，多个不同的值被提出来之后，最终只会有一个值被选定。如果一个值一旦被选定，这些进程能知道这个值。总结如下，为了保证safety的要求，算法要求:

    1.  只有一个值被提出之后才能被选定;
    2.  只有一个值最终被选定；
    3.  一个进程认为被选定的值必须是真的最终被选定的值。

在这个算法之中，有以下的概念: Proposal: 这里可以理解为代表了一个值，Proposer: 提出Proposal， Acceptor: 决定是否接受Proposal， Learner: 接受被选定的值。

### 选定一个值

 由于算法要求只有一个进程提出一个值的情况下这个值也会被选定，所以算法必须满足以下的要求:
```
  P1: 一个Proposer必须接受它收到的第一个Proposal
```

这又导致另外一个问题，由于系统中可能存在多个的Acceptor，这种做法可能导致不同的Acceptor接受了不同的值，所以这里最初一个规定:
```
  规定: 一个Proposal只有在一半以上的Acceptor接受的情况下才能被选定
```

  为了实现这个规定，就要求可以选定不同的Proposal(因为不这样的话就可能无法到达半数以上的Acceptor接受)。在这里，为了区分这些Proposal，我们赋予每个Proposal递增的一个编号。为了保证选定了不同的Proposal也能得到最终准确的结果，这里要求被选定的不同的Proposal的值是相等的。所以有如下的要求:
```
  P2: 如果一个Proposal被选定了，每个被选定的有更高的编号的Proposal的值必须与此相同
```

  为了选定一个Proposal，必须要求有一个Acceptor接受，也就是说:
```
  P2a: 如果一个Proposal被选定了，那么每一个被Acceptor接受的Proposal必须与此相同
```

这个算法中，通信是不可靠的，进程也可能失败后又重启，也就可能存在下面这种情况: 一个Acceptor c之前没有收到过之前的Proposal，又有一个"新"(可能从失败之后恢复了过来)的Proposer向其发送了一个有更高编号的带有不同值的Proposal，由于要求P1，c必须接受这个Proposal，这就会导致维持P1和P2a直接的矛盾。为了解决这个问题，提出了以下的要求:
```
 P2b: 如果一个Proposal被选定了，那么之后的Proposer提出的编号更高的Proposal的值也必须与此相同。
```

从P2b可以推导出P2a，P2a有可以推导出P2(具体证明略，可以参考[1]). 那么如果保证P2b呢，这里采用的方法就是P2c:

```
  P2c: 对于任意的v和n，如果一个值为v，编号为n的Proposal被提出，存在一个半数以上的集合S满足下面两个中的任意一个条件:
    a: S中的任意Acceptor没见接受过比n小的Proposal；
    b: S中的所有Acceptor接受过的最大编号的Proposal中，值为v。
```

.

### Proposal生成

   为了保证P2c，一个Proposer想要提出一个编号为n的Proposal必须要知道在多数Acceptor以及接受or将要被接受的编号小于n的Proposal的且有最高编号的Proposal。知道过去的情况是比较简单的，但是这里未来的情况是不能预测，所以Proposer不会尝试预测未来，而是要求Acceptor不会接受任何编号小于n的Proposal，这样就可以得到以下的Proposal提出算法：
  1. 一个Proposer选择一个新的编号 n，然后向一个半数以上的Acceptor的集合发送请求,要求:
       a.  Acceptor不会在接受编号小于n的Proposal；
       b. 如果Acceptor以及接受过Proposal，那么就向Proposer响应已经接受过的编号小于n的Proposal。
          这里的请求称为prepare请求。
  2. 如果一个Proposer收到了半数以上的Acceptor的响应，那么这个Proposer就可以生成编号为n值为所有响应中编号最大的Proposal的值，如果都没有值，那么就由Proposer自己决定。然后发送给半数以上的Acceptor的集合(1,2中的Acceptor的集合不要求相同。这里的请求称为accept请求。

### Acceptor接受Proposal

   Acceptor如何处理呢。可以发现这里有两种类型的请求，prepare请求和 accept请求。一个Acceptor和忽略任何请求而不会破坏算法的安全性。那么Acceptor上面时候可以接受请求和，这里做出如下的要求:
```
 P1a: 一个Acceptor接受一个编号为n的Proposal只能在它没有响应过任何的编号小于n的prepare请求。
```

 Acceptor可以直接忽略编号小于它已经响应过的prepare请求的prepare请求，也可以忽略一个已经被接受的Proposal的prepare请求。对于Acceptor，它要记得它已accepted的编号最大的Proposal，已responded的请求的Proposal，及时它失败重启。

### 算法描述

 到这里算法的过程就比较清晰了，算法的过程也就直接给出论文[1]中的原文(避免翻译变味):
```
 Phase 1. 
  (a) A proposer selects a proposal number n and sends a prepare request with number n to a majority of acceptors. 
  (b) If an acceptor receives a prepare request with number n greater than that of any prepare request to which it has already responded, then it responds to the request with a promise not to accept any more proposals numbered less than n and with the highest-numbered proposal (if any) that it has accepted. 

Phase 2. 
  (a) If the proposer receives a response to its prepare requests (numbered n) from a majority of acceptors, then it sends an accept request to each of those acceptors for a proposal numbered n with a value v, where v is the value of the highest-numbered proposal among the responses, or is any value if the responses reported no proposals. 
  (b) If an acceptor receives an accept request for a proposal numbered n, it accepts the proposal unless it has already responded to a prepare request having a number greater than n. 
```

.

### Learner

  一个值被选定之后，就想要发送给Learner，这里的方法是整个算法中任意理解的部分。一般来说有以下几种方法:
           1.  一个Acceptor接受一个Proposal就把值发送给所有的Learner;
           2.  把值发送给一部分Learner，如果这些Learner发送给其它的Learner；

由于信息会丢失，一个值可能没有被Learner接受到，Learner可以直接询问Acceptor，但是Acceptor可能失败导致无法获取到这个消息，如果Learner想要得到这个value，可以用上面描述的算法发出Proposal。

### 改进

  上面描述的算法存在这样的情况，2连个Proposer交替提出编号递增的Proposal，会导致算法进入死循环。这种情况下使用的解决方案是 选择一个特殊的Proposer，如果这个Proposer与半数以上的Acceptor成功沟通，如果它使用一个的编号为n的Proposal大于了之前的，那么它的Proposal会被获得接受。

## 参考

1. Paxos Made Simple: http://research.microsoft.com/en-us/um/people/lamport/pubs/paxos-simple.pdf
2. Paxos: https://en.wikipedia.org/wiki/Paxos_(computer_science)
