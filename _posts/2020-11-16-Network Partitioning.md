---
layout: page
title: Fault Tolerance for Partial Network Partitioning
tags: [Network, Distribution]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## An Analysis of Network-Partitioning Failures in Cloud Systems

### 0x00 基本内容

 这篇Paper研究了限制的一些分布式软件系统的Network-Partitioning的容错能力。网络分区在目前的环境中，一般要被认为是会发生的，这样事就是CAP中的P总是会有的，很多系统就变成了C和A之间的取舍。这里将Network-Partitioning分为complete network partitioning、partial network partitioning和Simplex network partitioning。第一种指一个部分的节点和其它的结点完全隔离开了，第二种指的是一部分结点之间直接的连接不通了，但是还可以通过其它部分的连通。Simplex network partitioning则指的是一个结点可以发送消息到另外一个，另外一个发送的消息这个结点确收不到，这种在UDP这样的协议下会发生，很多是因为路由故障的原因。现在的很多分布式系统都理论上有处理网络分区的功能，这篇Paper则开发了一个NEAT的系统，对一些系统进行了世界上的测试。发现大部分的系统在处理Network-Partitioning问题上或多或少都有一些问题。

### 0x01 General Findings

 针对多个分布式系统的测试发现了这样的一些问题：约80%的Network-Partitioning导致的故障都有一个catastrophic，灾难性的影响。有27%导致了数据丢失。比如MongoDB在网络分区的时候，连接primary replica的client可能读取脏数据，即发生了dirty read。90%的问题都是silent的，另外一些会导致很难处理的警告信息。即使系统返回了一些warnings的消息，很多情况下也不能很好的区分这些warnings消息的原因。21%的复制导致了永久性的隐患。比如RabbitMQ一个结点在发生Network-Partitioning故障的时候，它会认为其它部分故障了，自己作为单独的一个部分运行。后面即使网络恢复了也不会系统也不会恢复到之前的状态。

* Leader election, configuration change, request routing, 和 data consolidation这些部分是最容易出现问题的。常见的问题有old leader在被隔离之后，其写入可能失败了，但是读取可能仍然提供服务，这样会有读取到过期数据、脏读的问题。一些系统使用的选举leader的策略也有问题，比如选择log最长的，比如VoltDB。选择最后操作时间戳最新的，比如MongoDB。选择ID最小的，比如Elasticsearch，一些情况下可能导致数据丢失，

  ```
  0% of leader election failures are caused by electing a bad leader. This is caused by using simple criteria for leader election... . These criteria can cause data loss when a node from the minority partition becomes a leader and erases all updates performed by the majority partition.
  ```

  MongoDB中使用的为结点设置一个成为leader的priority可能导致选举不出leader的问题。另外在data consolidation的问题上，Redis, MongoDB等这样使用的write with the latest timestamp wins 或者是 the log with the most entries wins策略，可能导致数据丢失。这些策略没有检查副本和操作的状态，可能被覆盖的已经返回给client的。ZooKeeper中也发现了几个data consolidation问题，比如ZooKeeper其使用两种方式来同步结点之间的数据：一种是storage synchronization，用于同步大量的数据，另外一种in-memory log synchronization用于同步少量的数据。网络分区方式的时候，一个结点被隔离没有得到最近的更新的时候，恢复的时候使用storage synchronization来同步数据。这里的一个问题就是storage synchronization没有更新in-memory log。如果其后面变成Leader，另外的结点使用in-memory log synchronization同步leader的数据，就可能造成数据丢失。另外在一些request routing中，在有重试的时候可能导致一些操作被重复执行，比如Elasticsearch使用的策略。

* 有64%的故障发可能不需要client的访问，也可能只需要client访问分区的一部分。由于可能有并发的在部分区域的写入，这样的一些结果就是可能导致data conflicts。比如HBase的例子， region server处理client写入请求的时候，将log写入到HDFS中，log到一定大小的时候，会切换到新的log。如果一个网络分区的故障导致了一个region server和HMaster隔离，但是没有和HDFS隔离。HMaster会认为这个region server已经故障了，然后将region logs转移给其它的节点。这个时候如果原来的region server创建了一个新的log，HMaster不会注意到这个新的log，也不会将其赋给一个region server。这样保存在原region server上面的操作就会丢失了。这里可以看出，网络分区的时候，压迫考虑到所有的系统操作的容错处理，包括client的请求和内部的操作。网络分区故障的比例中，Paper中的数据是69%的网络分区是完全分区，29%的是partial partitions，2%是simplex network partitioning，后者一般是基于UDP的heartbeat的问题。另外99%的只分为了两个分区，1%分为2个以上。

* 对于Partial Network-Partitioning故障，大部分的系统都没有区分Partial Network-Partitioning和Complete Network-Partitioning，实际的系统中这两种故障导致的结果差别不大。容忍的Partial Network-Partitioning也是很麻烦的事情，比如可能导致节点之间对服务的正常运行情况有不同的认识，这样会导致一部分结点按照正常的逻辑运行，另外一部分按照处理异常的逻辑运行。比如MapReduce和Elasticsearch的例子。在MapReduce中，如果AppMaster和Resource Manager网络隔离，而两者都能连接到cluster nodes。这样会导致AppMaster执行完任务返回，而Resource Manager认为AppMaster复制，重新起来一个AppMaster重新运行。Elasticsearch的问题和MapReduce的问题类似。另外比如在HDFS中，如何client和一个Rack隔离，而NameNode可以访问这个Rack。这样会导致client的写操作失败，client然后想NameNode请求不同的副本，而NameNode可能返回同一个Rack，导致一些不可用的情况。另外在MongoDB和Elasticsearch的Leader选举中。在MongoDB中，使用了一个arbiter进程来认定谁是leader，而如何有两个结点能够连接这个arbiter进行，而两者之间是不通的，则可能导致Leader反复变化，造成系统的不可用。Elasticsearch的问题类似。RethinkDB 和 Hazelcast配置变更也会因为Partial Network-Partitioning导致问题。在RethinkDB中，移除一个节点的时候，如果有Partial Network-Partitioning，对于有五个副本 (A, B, C, D, 和 E)的系统，如果AB和DE隔离，但是两部分都可以访问C，如果D收到一个将副本改为2个的请求，则D将会移除A、B和C。C这个时候会删除自己的log，A和B对这个配置变动操作不清楚。C的log已经删除，改变配置的请求也一起删除，C这个时候就会响应A和B的请求，从而造成变成了两个部分。Hazelcast中也有类似的问题，其删除一个节点之后会删除本地的数据，如果这个时候部分网络分区隔离来新的primary，另外一个副本会将其推举为新的primary。如果central master能够联通两个部分，则会拒绝掉推举的请求，而且让其进行step down操作，会删除本地数据，如果在尝试从priamry下载。如果这个时候primary永久故障，则会导致数据丢失。

在Failure Complexity上，Paper中分析了故障的表现的变化过程，输入事件顺序带来的影响，网络故障的特点，事件上的限制以及系统规模等的多个方面。发现大部分的复制只需要deterministic的， 只需要三个or更少的输入事件(input events)，而且三个节点就可以复现。在数据的总结中，发现了：1. 83%的网络分区出发的故障需要三个or更少的input event来触发，这些input events在网络分区的情况下可以导致failures的比如Redis中的两个结点之间的sync操作的时候，可能导致接受结点的data log永久的损坏，比如RabbitMQ中在部分分区故障的时候有被old leader收到变成follwer的通知的时候，启动一个follower线程却没有停止leader线程导致的系统hang住等。对于events，有87%要求多个event以一种特定的顺序发生，而一般(84%)开始的事件就是有网络服务的事件。这样对于重现一个failure的话还是比较困难的，但是由于需要的events数量并不多，如果故障持续时间长一点的话就会在实际系统中比较容易出现。另外一个值得关注的问题是有大部分的故障可以由隔离了一个节点就触发(88%)。从隔离结点的角色上，45%是隔离任意一个数据副本，而剩余的是要隔离一个特点的结点 or 服务，比如leader replica或者是central services等。所以实际的系统测试隔离这样的特殊结点对于提高可靠性有比较大价值。在另外一些问题上：

*  在Timing Constraints上，有62%的情况是deterministic，即遇到对应的input events，故障就会发生。另外18%时间上的限制是可知的，比如几个heartbeat的时间，或者是等到之前leader下线的时间。另外有13%无法明确获取时间上的限制，7%左右是nondeterministic的，一般是并发操作导致的不确定性。
* 对于解决这些问题的方法，有47%要求重新设计系统的相关机制，比如MongoDB的leader选举协议(根据最新操作的时间戳)，Elasticsearch的配置变更协议。很多是系统开始设计的时候就没有思考如何比较好处理这些网络分区的问题。有些系统则直接说明会有网络分区导致的数据丢失问题，比如Redis，RabbitMQ则说明locking操作不能容忍网络分区，而Hazelcast提供的是best effort consistency，实际上可能有数据/操作丢失(很多情况下best effort xxx == no guarantee on xxx at all)。
* 而对于如何测试发现这些问题的时候，其实一般情况下三五个结点就可以了，甚至可以直接使用虚拟机模拟。

### 0x02 NEAT Framework

  在上面的发现总，这里总结出：一般设计者没有重视网络分区给系统带来的影响，另外也缺乏一种好用的网络分区故障测试工具。针对后面一种的情况，这里提出的解决方式是NEAT Framework，使用OpenFlow和iptables来实现，是用来不到2k行的代码。Paper中给出的一个测试Elasticsearch数据丢失的例子: 在第7行长发一个网络分区，隔离了primary s1，client1和s2，client2。但是这些结点和s3联通，s2发现primary到达不了的时候启动一个新的leader选举，s3投票给s2，这样就会导致出现两个leader。此时两边的写入都可以成功，而网络分区恢复之后，s2重新发现s1。Elasticsearch有会是的ID最小的成为leader，这样s1成为leader之后s2同步数据，然后导致写s2的数据丢失。

```java
1 public static void testDataLoss(){
2 	List<Node> side1 = asList(s1, client1);
3 	// other servers and clients in one group
4 	List<Node> side2 = asList(s2, client2);
5 	// create a partial partition. s3 can reach
6 	// all nodes
7 	Partition netPart = Partitioner.partial(
8 		side1, side2);
9 	sleep(SLEEP_LEADER_ELECTION_PERIOD);
10	 // write to both sides of the partition
11	 assertTrue(client1.write(obj1, v1));
12	 assertTrue(client2.write(obj2, v2));
13	 Partitioner.heal(netPart);
14	 // verify the two objects
15	 assertEquals(client2.read(obj1), v1);
16	 assertEquals(client2.read(obj2), v2); 
17}
```

NEAT Framework提供的接口如下，基本就是创建其三种不同类似的网络分区，以及修复分区。在具体的网络连通与断开上，使用iptables和OpenFlow来实现，其都可以通过添加一些规则来drop掉那些网络数据包。

```java
Partition complete(List<Node> groupA, List<Node> groupB);
Partition partial(List<Node> groupA, List<Node> groupB);
Partition simplex(List<Node> groupSrc, List<Node> groupDst);
void heal(Partition p);
```

### 0x03 总结

  这里的具体信息可以参看[1].

## Toward a Generic Fault Tolerance Technique for Partial Network Partitionin

### 0x10 Dissecting Modern Fault Tolerance Techniques

 这篇Paper是前面一篇的后续，内容中有很多相同的地方，重复的部分直接忽略了。这篇Paper的主要内容之一就是讨论Partial Network Partitionin的问题。如何处理这里先总结了几个分布式系统处理网络分区的机制。Paper中分析的很多分布式系统中有很多也添加了处理 Partial Network Partitionin的机制。

#### Identifying the Surviving Clique

在 Partial Network Partitionin的情况下，一些系统处理的主要思路是找到一个最大的clique of nodes，即最大的一个完全连接的系统结点的一部分，其它的部分就直接下线处理。VoltDB处理的基本思路就是这种：在VoltDB中，每个结点周期性地向其它所有的结点发送heartbeat消息。如果一个结点和任何的一个结点失去连接，则就怀疑网络有了分区故障。开始进行入恢复的处理机制，这个机制分为两个操作步骤：1. 发现有失去连接的结点向其它可达的结点发送这个消息，2. 其它结点在收到之后，综合每个结点的可达性信息，得出一个可达结点的一个图。每个结点根据合并的消息得出最大的clique of nodes信息。然后发现自己不在这个clique中的时候，就进行下线的操作。在剩余的结点中，系统分析它没有丢失任何的data shard。如果发现有data shard(s)所有副本都下线的情况，则整个系统都会shout down，从而变得不可用。Paper中分析了这种策略的缺点：1. 其没有必须关闭一部分的结点(最多一半)，这样会带来性能和容错能力的下降；2. 在一个shard的所有副本都下线的情况下会导致整个系统不可用，实际上下线20%的结点就有90%的可能导致系统完全不可用：

```
Each shard has three replicas. Our analysis shows that isolating only 10% of the nodes leads to more than a 50% probability of shutting down the entire cluster, and isolating only 20% of the nodes leads to a staggering 90% chance of a complete cluster shutdown.
```

#### Checking Neighbors’ Views

 另外一种方式称之为Checking Neighbors’ Views，基本思路是一个结点S发现另外一个结点D不可达的时候，询问其它结点是否可达D。如果有可达的，则认为发生了Partial Network Partitionin的情况。S会断开所有的和可达D结点的连接，这样会将Partial Network Partitionin变成Complete Partition。或者是暂停其的操作。RabbitMQ 和 Elasticsearch 使用这样的方式：

* 在RabbitMQ中，认为发生了Partial Network Partitionin情况之后，根据系统配置可以选择这样的处理策略：1. Escalate to a complete partition，即基本思路中将部分网络分区变成完全网络分区的策略。这种方式可能导致数据不一致，在恢复之后需要数据合并操作(data consolidation mechanism)。2. Pause，暂停操作的策略是为了避免出现数据的不一致。这样系统会只有一部分的完全连接的结点可以进行操作；3. Pause if anchor nodes are unreachable，这种策略和第二种的区别是：这种情况下配置了一部分特殊的结点作为anchor nodes，如果一个结点到达不了其中任一结点，这个结点就会暂停工作。在anchor nodes部分隔离的情况下，这种方式可能导致多个complete partitions。Anchor nodes结点被隔离的情况下，会造成所有的结点停止服务。恢复之后使用administrator intervention来合并数据。这些策略的确定是：1. 将部分分区变成完全分区的策略可能导致数据出现 multiple inconsistent copies。而pause的策略可能导致整个系统不可用，比如当出了结点1外所有结点都发现了不可达结点，但是结点1都可达，则会造成整个系统不可用。
* Elasticsearch中的策略是一个节点S发现其不可达Master时，如何其它结点告诉它，它们可以大，则S会暂停自己的操作。如果没有结点可到Master，ID最小的成为新Master。这种方式的确定是：结点之间可能不能达成Master不可达的共识，比如出节点2外，其它结点到达不了节点1。这样2就会拒绝成为新的Master，其它结点都会暂停工作。而Master不能连通半数以上节点，其也会暂停工作。从而导致系统不可用。

### Failure Verification

 这里处理另外的一个问题是Failure Verification，即确认故障。基本的思路一般是：一个节点S收到其它节点其不能达到另外一个节点D时候，S会先验证它能达到D吗。这个也是很多操作的第一步。比如在MongoDB 和 LogCabin中，如果一个leader在一个部分分区中，能达到的节点数量超过半数，会导致另外的分区开始一次不需要的选举，这个可能导致持续的leader选举从而影响到系统的可用性。为了处理这个问题，一个节点在收到选举的消息之后会先确认其能否到达leader。这种方式有两个主要的问题：一个是可能导致不少的结点无法使用，另外一个是其机制是一个特定的机制，没有很好的通用性。

#### Neutralizing Partitioned Nodes

 处理部分网络分区一个比较麻烦的地方是由于有些节点可以接受来自不同部分的更新请求，从而造成数据都是 or 数据不一致。为了处理这个问题，在HBase, MapReduce, 和 Mesos 采用了Neutralizing的思路。即通过一些机制来避免出现这种情况。以HBase为例，如果HBase leader无法连接一个HBase node，它通过对应data shard在HDFS中的目录进行重命名，这样避免旧结点对其进行更新。又比如在在MapReduce中，在部分网络的情况下，可能导致两个AppMaster。为例处理这个问题，AppMaster在完成一个task之后会写一个完成记录到一个HDFS上的shared log中，来进行标记。这样的方式缺点是对system-wide fault tolerance不太可行，往往是处理一个特点的问题。另外可能导致一部分资源被浪费，降低系统性能。

### 0x11 Nifty

 Paper中给出了一个有通用性的部分网络分区解决方式，即在部分网络分区分区的时候，通过那些可以达到所有分区的结点来进行数据转发，这样结点称之为 bridge nodes。Nifty使用heartbeat探测和其它所有进程的连接性，同时维护一个和其它进行的distance vector，单位为跳数。如果发现了3个以上的heartbeat丢失，则认为连接不通，但是还是会继续发送heartbeat消息。每个Nifty会发送自己的distance vector给其它的Nifty进程，其它进程利用这个来构建一个routing table，类似于一些记录距离向量的路由协议，比如RIP。在实现这样的思路时候，使用 OpenFlow 和 Open vSwitch来实现新的路由路径。

### 0x12 评估

 这里的具体信息可以参看[2].

## 参考

1. An Analysis of Network-Partitioning Failures in Cloud Systems, OSDI '18.
2. Toward a Generic Fault Tolerance Technique for Partial Network Partitioning，OSDI ‘20.

