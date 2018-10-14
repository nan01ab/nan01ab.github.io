---
layout: page
title: Dynamo -- Amazon’s Highly Available Key-value Store
tags: [Storage, Distributed]
excerpt_separator: <!--more-->
typora-root-url: ../
---



## Dynamo: Amazon’s Highly Available Key-value Store 



### 引言

  Dynamo是分布式Key-Value系统的经典设计。Dynamo为了追求高可用和可拓展，适当地放弃了一致性，对之后的很多系统的设计产生了很大的影响。对于Dynamo有以下几个系统假设和要求：

* Query Model，简单的接口，一般就是get，put之类的key-value接口；
* ACID Properties，ACID (Atomicity, Consistency, Isolation, Durability)中只支持弱一致性，也不提供隔离性的保证，只允许单key的更新；
* Efficiency，要求低延时，高吞吐；

```
Other Assumptions: Dynamo is used only by Amazon’s internal services. Its operation environment is assumed to be non-hostile and there are no security related requirements such as authentication and authorization. Moreover, since each service uses its distinct instance of Dynamo, its initial design targets a scale of up to hundreds of storage hosts. We will discuss the scalability limitations of Dynamo and possible scalability related extensions in later sections.
```

.

### 特点

下面的表格总结了Dynamo要解决的问题，已经解决这个问题使用的技术和所具有的优点：

![dynamo-techniques](/assets/img/dynamo-techniques.png)

 

### 接口

 Dynamo只提供很简单的key-value接口，`get(key)`接口返回这个key对应的对象or对象链表，`put(key, context, object)`接口决定这个key对应的对象副本应该被保存在哪里。`context`参数里面包含的是系统的元数据，这些事不透明的。



### 分区算法

  Dynamo为了支持可持续的拓展，要求一种在一组节点上面动态分区的方式。Dynamo使用的基本方式就是一致性hash。一般的一致性hash的算法存在2个问题，一个是可能造成的发布不均匀的问题，另外一个是不能察觉结点之间性能的差异；

  Dynamo为了解决这些问题，使用了一种一致性hash的变体，就是一个时间的物理结点代表了多个的虚拟结点(这个不是一种很常见的方式吗？？)，一个特点是一个结点可以代表的虚拟结点可以根据这个结点的特点决定，也就是说，一个性能更加好的结点可以负责更多的虚拟结点。

```
Using virtual nodes has the following advantages:
• If a node becomes unavailable (due to failures or routine maintenance), the load handled by this node is evenly dispersed across the remaining available nodes.

• When a node becomes available again, or a new node is added to the system, the newly available node accepts a roughly equivalent amount of load from each of the other available nodes.

• The number of virtual nodes that a node is responsible can decided based on its capacity, accounting for heterogeneity in the physical infrastructure.
```

分区的方式要处理的一个问题就是成员的加入和退出。Dynamo新加入一个结点X时，它会随机为其选择一个环上的范围。由于在加入X之前处理这个范围的结点存在多个，在X加入之后，有些结点就不需要处理这个范围内的数据了，这些结点将会将这些数据传输给X。对于删除的情况，就是加入操作的反操作。

.

### 复制

  复制是一种实现high availability一种很常见的方式。Dynamo中的复制的办法是系统配置了一个复制数量的参数N，处了在这个key对用的结点保持一份之外，还做它的N-1给后继结点保存一份(这些结点组成了一个环，用于一致性hash)。保存一个key的这些结点的list叫做preference list。为了解决使用虚拟结点带来的可能一个物理结点代表了对个位置的问题，这里这个list只会保留不同的结点。

````
To account for node failures, preference list contains more than N nodes. Note that with the use of virtual nodes, it is possible that the first N successor positions for a particular key may be owned by less than N distinct physical nodes (i.e. a node may hold more than one of the first N positions). To address this, the preference list for a key is constructed by skipping positions in the ring to ensure that the list contains only distinct physical nodes.
````

.

### 数据版本

  Dynamo支持的是弱一致性。一个对象的更新在各个副本上是异步进行的，所以get操作可能返回的不是最新的数据版本，正常情况下，更新操作传播的是有上限的，但是在故障的情况下，这个时间就变得不可知。Paper给出了一个购物车的例子，要求Dynamo能够处理多个版本的数据，Dynamo这里使用的是每次修改操作都会生成一个不可变的数据版本，它还允许系统中同时出现多个版本的数据。正常的情况下，旧版本的数据会被归入到新的版本中，系统可以决定其中的一个版本是权威性性的版本。这里说了正常的情况就因为着有不正常的情况，在出现故障加上并发更新的情况下，就有可能出现版本冲突，这个时候系统就不能处理这个问题了，需要客户端协调处理。Paper中的例子还是购物车的例子，这个冲突可能造成被删除的物品重新出现在购物车中，

```
 In these cases, the system cannot reconcile the multiple versions of the same object and the client must perform the reconciliation in order to collapse multiple branches of data evolution back into one (semantic reconciliation). A typical example of a collapse operation is “merging” different versions of a customer’s shopping cart. Using this reconciliation mechanism, an “add to cart” operation is never lost. However, deleted items can resurface.
```

 Dynamo是用vector clock来处理一个对象不同版本之间的因果关系(关于vector clock的细节信息可参考相关资料)，通过检查vector clock就可以判断两个版本指甲是平行的关系还是因果的关系。

```
If the counters on the first object’s clock are less-than-or-equal to all of the nodes in the second clock, then the first is an ancestor of the second and can be forgotten. Otherwise, the two changes are considered to be in conflict and require reconciliation.
```

Dynamo的客户端想要更新一个对象的时候必须指定这个对象的版本，这里之前提到了`put`操作的接口中包含了一个context的参数，这个参数重就包含了vector clock的信息。当处理一个读的请求时，Dynamo如果发现了数据版本的多个分支，它自己时不能处理，而是会将这些数据都返回给客户端，同时返回它们的context信息。然后使用这个context更新这个对象的操作将会被视为已经协调好了不同的版本分支之间的问题，这些将会被合并为一个。下面这幅图就表示了这个过程.

![dynamo-version-reconcile](/assets/img/dynamo-version-reconcile.png)



 此外，这个vector clock的大小时不太可能增长到很大的，但在网络分裂之类的故障下，还是存在这个可能的，所以Dynamo处理了这种情况，使用的方法就是vector clock重的`(node, counter) `对增长到一定数量时，就把最后的一项去掉，这个会对系统产生什么样的影响还没有被彻底研究，但是没有出现过问题。

```
 When the number of (node, counter) pairs in the vector clock reaches a threshold (say 10), the oldest pair is removed from the clock. Clearly, this truncation scheme can lead to inefficiencies in reconciliation as the descendant relationships cannot be derived accurately. However, this problem has not surfaced in production and therefore this issue has not been thoroughly investigated.
```

.

### `get ()` 和 `put ()` 执行

   Dynamo的任何结点都可以处理任何key的get put操作(没有什么master之类的)。Dynamo的请求一般使用的是HTTP的方式，client可以向一个load balancer发送请求，也可以直接处理分区的客户端直接亲戚合适的协调结点(coordinator nodes)。这个coordinator一般就是preference list中的第一个结点。如果请求时通过load balancer放送，则这个请求可能被路由到环中的任意结点，如果这个节点不在这个key的preference list中，则它会转发这个请求到preference list中的第一个结点。

  为了保持数据的一致性，Dynamo使用了一种类似quorum的机制，这里就可以参考关于quorum的相关资料：

```
This protocol has two key configurable values: R and W. R is the minimum number of nodes that must participate in a successful read operation. W is the minimum number of nodes that must participate in a successful write operation. Setting R and W such that R + W > N yields a quorum-like system. 
```

Coordinator结点收到put请求的时间，它就会为这个新版本生成vector clock，并将这些数据发送给preference list中前N个可达的结点，当有W - 1个结点返回成功时这个操作就算成功了(加上自己就是W个)，同理get请求则是等待R个结点返回。



### 故障处理

#### Hinted Handoff 

  前面说了Dynamo采用了类似quorum的处理机制，但是这个可能影响到系统的可用了，为了解决这个问题，Dynamo使用一个叫做`sloppy quorum`的方式。这种方式使得读写操作可用是环上preference list前面N个健康的结点而不一定就是前N个。这里如果中间跳过的结点之后恢复了的话(假设为A)，做为补充的结点(假设为D，因为跳过了A，它就成为了前N个中的一个)，D就会尝试将之前的数据发送回这个恢复过来的结点A。传输完成之后D就可以将这个数据删除了。这种方式在Dynamo中就被叫做`hinted handoff`。

```
Using hinted handoff, Dynamo ensures that the read and write operations are not failed due to temporary node or network failures. Applications that need the highest level of availability can set W to 1, which ensures that a write is accepted as long as a single node in the system has durably written the key it to its local store.
```

.

#### Handling permanent failures: Replica synchronization 

  上面的`hinted handoff`很好地处理了临时性的故障。为了处理故障结点没有会到环中造成的对可用性的影响，这里Dynamo使用的方式是一种`anti-entropy` (replica synchronization)协议来保持副本的同步。Dynamo使用了一种叫做`Merkle Tree`的方法来快速检测副本之间的不一致、并减少传输的数据量(这里关于Markel Tree可以参考相关资料),

```
For instance, if the hash values of the root of two trees are equal, then the values of the leaf nodes in the tree are equal and the nodes require no synchronization. If not, it implies that the values of some replicas are different. In such cases, the nodes may exchange the hash values of children and the process continues until it reaches the leaves of the trees, at which point the hosts can identify the keys that are “out of sync”.
```

Dynamo使用Merkle Tree的方式是：每一个结点为一个key的范围维持一个Merkel Tree(这个范围就是一个虚拟结点覆盖的key的范围)。这可以使得结点之间可以快速比较在这个范围的数据是否一致，并执行适当的同步操作。这种方式的一个缺点就是在结点离开or加入环的时候，这些tree都得重新计算，Dynamo通过改进partitioning解决了这个问题，具体可以参考论文。



### 成员及其故障探测

  为了处理成员变化的各类情况，Dynamo使用了人工处理成员变化的方式。一个基于gossip的协议用来传播协议变化的信息。

```
A gossip-based protocol propagates membership changes and maintains an eventually consistent view of membership. Each node contacts a peer chosen at random every second and the two nodes efficiently reconcile their persisted membership change histories.
```

此外，结点在环上的映射信息已经分区的信息也是通过基于gossip的协议来传输的。每一个结点都知道另外的结点处理的范围，这个使得每个结点可以正确处理key的读写操作。

 Dynamo另外要处理的一个问题就是`logically partitioned `。它在这样一种情况出产生，管理员先节点A加入到环，然后将节点B加入环。在这种情况下，A和B都认为自己是环的一员，但是会存在Ａ不知道B的存在，B也不知道A的存在的情况，这个就叫做`logically partitioned `。这里的处理方式是使一些结点为`seeds`角色，它使用外部的机制发现所有的结点(external mechanism)，这里就可以理解为一些结点不是通过常规的方法获取到这些信息的，而是有另外的来源，一个常见的策略就是在某个地方保存了相关的配置信息。

```
To prevent logical partitions, some Dynamo nodes play the role of seeds. Seeds are nodes that are discovered via an external mechanism and are known to all nodes. Because all nodes eventually reconcile their membership with a seed, logical partitions are highly unlikely. Seeds can be obtained either from static configuration or from a configuration service. Typically seeds are fully functional nodes in the Dynamo ring.
```

关于故障检测，Dynamo也是用来基于gossip的去中心化的协议。

.

### 评估

性能表现:

![dynamo-performance](/assets/img/dynamo-performance.png)



## 参考

1. Dynamo: Amazon’s Highly Available Key-value Store, SOSP'07.