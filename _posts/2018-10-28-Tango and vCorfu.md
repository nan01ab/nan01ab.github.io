---
layout: page
title: Tango and vCorfu -- Tow Papers about Shared Log
tags: [Distributed]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## Tango: Distributed Data Structures over a Shared Log

### 0x00 引言

 Tango这篇Paper是利用Shared Log实现分布式数据结构的一篇文章。 Tango可以实现类似于C++ STL / Java Collections中的数据结构。这些数据结构的抽象就类似于一般的内存中的数据结构，但是会备份到Shared Log中，

```
... a single Tango object running on 18 clients provides 180K linearizable reads/sec against a 10K/sec write load... In our evaluation, a set of Tango objects runs at over 100K txes/sec when 16% of transactions span objects on different clients.
```

### 0x01 基本架构

  Tango中的一个Tango对象是抽象的一个核心，它是一个在Shared Log中保存数据结构的一个副本，Tango使用两个基本的API来方便实现对对象的更新和查询，

```
• update helper: this accepts an opaque buffer from the object and appends it to the shared log.
• query helper: this reads new entries from the shared log and provides them to the object via an apply upcall.
```

 对于一个Tango对象来说，它包含了3个主要的组件： 1. View，这个是一个Tango对象在内存中的表示，如list，map等；2. 一个apply方法，当来自Shared Log的新的日志项时被Tango的运行时调用，这个方法不应该被应用自身调用；3. 这个Tango对象对外暴露的外部接口。下面是一个利用Tango实现一个Register的例子，Tango对象的更新依赖于update_helper；而读取一个对象之前要执行query_helper方法，这个的目的是拉取最新的日志记录。由于所有的更新记录都保存在Shared Log中，Tango的回滚到过过去状态也很容易，只要重新创建一个实例并同步适当的Log的前缀即可。

```c++
class TangoRegister { 
   int oid;
   TangoRuntime ∗T;
   int state ;
   void apply ( void ∗X) {
      state = ∗(int ∗)X; 
   }
   void writeRegister(int newstate){ 
      T−>update_helper(&newstate, sizeof(int), oid );
   }
   int readRegister() {
       T−>query_helper(oid);
       return state ;
   }
}
/*
Figure 3: TangoRegister: a linearizable, highly available and persistent register. Tango functions/upcalls in bold.
*/
```

   Tango可以在一个Shared Log里面实现多个Tango对象，为例区分对象的日志项，每一个日志记录会包含一个OID(对象ID)，这样的一个缺点在于每一个客户端必须“播放”整个日志。在Tango中实现强一致性的操作更加简单，另外Tango还在Shared Log的基础之上实现了一种乐观的并发控制协议：基本的方式是在日志中追加一个*commit records*记录，每一个Commit的记录会包含一个读取对象的集合，并附上这些对象的版本的信息。只有在日志中遇到一个提交记录的时候这些读取对象的集合版本都没过过时的时候这个提交才会是成功的。Tango中事务的使用方式也和一般数据库的方法类型，以一个 *BeginTX*调用开始，这个会在一个线程局部存储中创建一个事务的上下文的信息， *EndTX*调用则是追加一个提交记录，然后决定这个事务是commit还是abort。每一个客户端遇到这样一个提交的记录之后，会独立的作出这个事务是否成功提交的判断，虽然是独立作出的判断，但是这个判断的结果会是确定的。

### 0x02 分层的分区

  为了解决前面的在一个日志里面保存多个Tango对象必须要求客户端“播放”所有日志项的缺点，Tango使用了一种*layered partitioning*的方法。每一个客户端会保存一份全局状态的分区，这些分区在不同的客户端可能是重叠的。这些分区的模式还是建立在一个单独的Log上面，但是Tango在上面抽象了一个Stream，多个Stream可以同时存在一个Shared Log中。这个Stream的抽象在一般的Shared Log的接口上面增加了一个readnext的一个接口，这个readnext的接口使得客户端可以直接读取这个Stream的下一个记录，而跳过属于其它的Stream的日志记录。一个*multiappend*可以使得一个日志项属于多个Stream。

* 在Stream的上面执行事务的时候，如果只是在一个对象上面的操作，那这件事情就是trivial的。如果跨越多个对象，这里就需要*multiappend*的方法来实现追加多个对象可见的commit的日志项，这里确认这个事务是否成功提交的方法基本上和前面的是一样的，

  ```
  The first time it encounters the record at a position X, it plays all the streams involved until position X, ensuring that it has a consistent snapshot of all the objects touched by the transaction as of X. It then checks for read conflicts (as in the single-object case) and determines the commit/abort decision
  ```

    不过这里另外的一个要处理的事情就是不是所有的数据都保存在一个客户端上面，所以在验证是否能提交的水货要处理不在本地的数据。这里具体分几种情况进行处理。

* Remote writes at the generating client，执行这个事务并追加commit记录的客户端想要区访问一个远程对象的时候只需要直接追加commit记录即可；

* Remote writes at the consuming client，一个不是执行这个事务的客户端遇到一个commit的记录，这个事务涉及到写入一个它没有保存的对象，这个客户端只用处理自己涉及到的对象，而不同区管其它的对象；

* Remote reads at the consuming client，一个不是执行这个事务的客户端遇到这个commit记录的时候，这个事务涉及到读取一个它没有保存的对象，这个客户端通过追加一个添加记录，并回放对应的日志知道这个事务的提交的点，然后决定是否提交，其它的时候在这种情况下就可以等待这个客户端作出决定。这个客户端作出决定之后会添加一个decision record日志记录。这样的操作的一个缺点是会显著增加延迟，但是不会增加abort的比例。

* Remote reads at the generating client，目前者Tango中这种情况是不被允许的。

* 错误处理，Tango的错误处理都是基于超时的，前面没有Stream抽象的操作也是一样。因为日志中包含了最初事务决定的所有信息，这里只需要超时后重新操作即可。

### 0x03 评估

 这里具体的信息可以参看[1],

<img src="/assets/img/tango-perf.png" alt="tango-perf" style="zoom: 67%;" />

## vCorfu: A Cloud-Scale Object Store on a Shared Log

### 0x10 引言

  vCorfu这篇Paper是使用Shared Log实现的一个分布式的对象存储系统。在vCorfu中，Stream的抽象不仅仅是一个Shared Log上面的标记而已，而是一个“一等公民”。vCorfu中称之为*stream materialization*。vCorfu中的操作也更加强调Stream的存在，

| 操作                             | 描述                                     |
| -------------------------------- | ---------------------------------------- |
| read(laddresses)                 | 获取保存在Log Address指定位置的数据      |
| read(stream, saddresses)         | 获取保存在Stream Address指定位置的数据。 |
| append(stream, data)             | 追加数据到一个Stream                     |
| check(stream)                    | 获取一个Stream的最后的地址信息           |
| trim(stream, saddresses, prefix) | 释放资源，stream address < prefix        |
| fillhole(laddress)               | 填充空洞                                 |

* vCorfu中日志追加的操作也是在Stream的维度进行操作，这个与前面的Tango的实现存在明显的区别。
* vCorfu的复制是在Log和Stream两个层面进行的，索引是不同的，这样获取一个Stream中最新的数据的时候只要从Stream的副本获取 即可。
* 在vCorfu中，一个layout服务器保存了Log和Stream两个层面的地址空间到副本之间的映射关系，每一个副本保存的更新的数据是不同的。Log副本保存全局的更新信息，而Stream副本只会保存这个Stream的更新的信息。Stream将一个全局的Shared Log划分为多个Stream。
* Layout表示了vCorfu中地址空间的划分，这里和Corfu的是很类似的。这些信息会使用一个基于Paxos的协议来确保所有的副本都一致同意这个Layout。和很多的系统中使用的Epoch方法一样，这个Layout也包含了一个Epoch 的信息，方便处理Layout的更新。

<img src="/assets/img/vcorfu-layout.png" alt="vcorfu-layout" style="zoom: 67%;" />

* 在Corfu中，Sequencer负责处理token只是Log的，在vCorfu中，则包含了全局的Log和指定Stream两个层面的Token。

<img src="/assets/img/vcorfu-write-path.png" alt="vcorfu-write-path" style="zoom:67%;" />

* vCorfu的materialized stream设计是的设计到多个Stream设计变得简单。为例原子地在多个Stream中追加数据，客户端先向Sequencer获取一个Log的Token和每个Stream的Token，然后先写Log的副本，再写每个Stream的副本，最后给每一个参与的副本发送提交的信息。
* 与前面的Tango的设计比，vCorfu设计使得GC也变得简单，而在Tango中不能直接trim一个Stream，在vCorfu可直接这样操作。

### 0x11 基本架构

  vCorfu运行时的设计也基本是借鉴了前面的Tango的设计，在vCorfu的实现中使用的是Java，而Tango使用C++。开发者可以直接在vCorfu中保存任意的Java对象。和前面的Tango一样，利用vCorfu也可以实现List，Queue和Map之类的数据结构。另外vCorfu也和Tango一样，vCorfu也有事务的支持。

<img src="/assets/img/vcorfu-arch.png" alt="vcorfu-arch" style="zoom:67%;" />

  vCorfu的事务的设计和Tango中的有所不同，它利用Sequencer作为一个轻量级的事务管理器，

* vCorfu事务使用的方法和一般的也差不多，也是以一个TXBegin调用创建一个事务的上下文，以一个TXEnd提交事务；
* vCorfu中也和Tango一样，事务中写入的数据保存在一个Write Buffer中。不同的是在提交的时候，如果vCorfu发现这个Buffer为空，则事务执行成功，为一个只读事务；
* 如果Buffer不空，则客户端通知Sequener它使用的Token，如果设计到的Stream没有更改，则Sequencer发放Token。客户端通过提交Write Buffer来提交事务。否则事务Sequencer不发放Token，事务会abort。这样的方法就可以确保在vCorfu中Log只有确认可以提交的日志，而不同和前面的Tango一样使用验证的方法。

### 0x12 可组合的复制状态机

  在vCorfu中，一个对象可能是其它的对象组成的，这个给开发带来了很多的便利。

```
... First, CSMR divides the state of a single object into several smaller objects, which reduces the amount of state stored at each stream. Second, smaller objects reduce contention and false sharing, providing for higher concurrency. Finally, CSMR resembles how data structures are constructed in memory - this allows us to apply standard data structure principles to vCorfu. 
```

下面是一个Java代码的例子， 

```java
 class CSMRMap<K,V> implements Map<K,V> {
    final int numBuckets;
     
    int getChildNumber(Object k) {
      int hashCode = lubyRackoff(k.hashCode());
      return Math.abs(hashCode % numBuckets);
    }
     
    SMRMap<K,V> getChild(int partition) {
      return open(getStreamID() + partition);
    }
     
    V get(K key) {
      return getChild(getChildNumber(key)).get(key);
    }
     
    @TransactionalMethod(readOnly = true)
    int size() {
      int total = 0;
      for (int i = 0; i < numBuckets; i++) {
        total += getChild(i).size();
      }
     return total;
    }
     
    @TransactionalMethod
    void clear() {
      for (int i = 0; i < numBuckets; i++) {
        total += getChild(i).clear();
      }
    }
 }
```

### 0x13 评估

 这里的具体信息可以参看[2],

<img src="/assets/img/vcorfu-perf.png" alt="vcorfu-perf" style="zoom:67%;" />

## 参考

1. Tango: Distributed Data Structures over a Shared Log, SOSP'13. 
2. vCorfu: A Cloud-Scale Object Store on a Shared Log, NSDI'17.

