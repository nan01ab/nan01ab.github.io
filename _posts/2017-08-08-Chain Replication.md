---
layout: page
title: Chain Replication
tags: [Distributed]
excerpt_separator: <!--more-->
typora-root-url: ../
---



## Chain Replication for Supporting High Throughput and Availability 

  Chain Replication是一个容易理解的复制方式，primary-backup，N个服务器可以容忍N-1服务器失效。它不处理拜占庭类的错误。

![chian-replication-chain](/assets/img/chian-replication-chain.png)

   上面这幅图就很容易说明了它的工作方式，更新操作发送给head，随着chain传递到tail，而查询等的操作都发送给tail。

​    在client看来:

![chain-replication-client-view](/assets/img/chain-replication-client-view.png)



### 错误处理

  在这里添加了一个master的服务，它用来探测服务器故障，通知服务器它的pre-decessor 和 successor 更新消息，通知client哪一个服务器是head，哪一个服务器是tail。Master自身使用Paxos来保证可靠性。



#### head 故障

   head的下一个服务器成为新的head。

```
PendingobjID is defined as the set of requests received by any server in the chain and not yet processed by the tail, so deleting server H from the chain has the effect of removing from PendingobjID those requests received by H but not yet forwarded to a successor. Removing a request from PendingobjID is consistent with transition T2, so deleting H from the chain is consistent with the specification in Figure 1.
```

,

#### middle-server故障

  master通知故障服务器s下一个服务器s+和s的上一个服务器chain的变化。这里要处理的问题稍微多一个点，因为中间服务器故障的话可能导致一些更新丢失。这里使用的方式是一个服务器保存一些状态消息，以便于它的下一个服务器更改是能够补充发送没有传递到后面的请求。	

```
This, however, could cause the Update Propagation Invariant to be invalidated unless some means is employed to ensure update requests that S received before failing will still be forwarded along the chain. The obvious candidate to perform this forwarding is S−, but some bookkeeping and coordination are now required.
```

,

#### tail 故障

 tail的上一个服务器称为新的tail。

```
 ... so changing the tail from T to T− potentially increases the set of requests completed by the tail which, by definition, decreases the set of requests in Pending objID . 
```

,

#### chain 拓展

   使用的方式就是将tail的消息拷贝过来之后，新的服务器称为新的tail。

```
Inprocess Requests Invariant is established and T+ can begin serving as the chain’s tail:
* T is notified that it no longer is the tail. T is thereafter free to discard query requests it receives from clients, but a more sensible policy is for T to forward such requests to new tail T

* Requests in SentT are sent (in sequence) to T+.

* The master is notified that T+ is the new tail.

* Clients are notified that query requests should be directed to T+.
```

.

### 其它

​     论文中还讨论了其它很对东西，比如这里还可以不只有一条链，可以有多条。故障之后对处理请求的影响，性能评估等等。

   总的来说Chain Replication是一个比较简单明了的 Replica方式，不过感觉不是很实用。



## 参考

1. Chain Replication for Supporting High Throughput and Availability, OSDI 2004.
