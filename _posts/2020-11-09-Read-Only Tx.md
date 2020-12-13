---
layout: page
title: The SNOW Theorem and Performance-Optimal Read-Only Transactions
tags: [Database, Transaction, Distribution]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## The SNOW Theorem and Latency-Optimal Read-Only Transactions

### 0x00 基本内容

 这篇Paper仿照CAP Theorem提出了一个SNOW Theorem。即S，N，O和W不能同时满，根据这里SNOW Theorem来知道系统的设计，Paper中也给出了两个分布式事务系统的例子，一个满足SNW，一个满足NOW。其中：

* S表示的是Strict Serializability，即保证全系统中的事物中，存在一个total order，全局的顺序。即事物是serializable的。另外Strict有要求了这个total order反应的是real-time ordering，即如果事务t2在real time中发生在事务t1已经完成之后，则t2在total order中必须出现在t1之后。这里的real-time应该是实际时间的意思，而不是Computer Science中常见的实时的意思。但是有另外的两个事务t3和t4，其中t4开始的时候，t3已经开始了但是还没有完成，在total order中哪个出现在前面都是可以的。
* N表示的是Non-Blocking Operations，即系统在执行Read-Only Transaction的时候，不需要等待任何的event，这个event可以是等待锁，等待其它server的消息，或者是等待超时等等。
* O表示的是One Response Per Read即client请求server读的时候，client只要发送一次的read请求，而server也只会回复一个的response。
* W指的是Write Transactions that Conflict，即可以有冲突事务同时进行，比如一个写事务更新多个server上的数据的时候，读事务还是可以进行的。

#### 证明

Paper假定的环境是一个N个Servers的分布式系统，N大于1。Client和servers通过网络通信，这个网络是asynchronous network，即message delay可以是任意时间的(partially synchronous network则可以是有delay的，不过这个delay存在一个上界)。这里定义 invocation time为一个clieng发送请求到参与这个事务的server的时间，response time定义为client收到所有server回复的时间。Lamport的happened-before关系定义了这样的关系：

```
1) If a and b are events in the same process, and a comes before b, then a → b. 
2) If a is the sending of a message by one process and b is the receipt of the same message by another process, then a → b. 
3) If a → b and b → c then a → c.
```

对于一个server上来自一个read-only事务R的一个操作r，有另外一个write事务，client发出r的时候其不知道W的存在，server在接受到W的请求之前也不知道W的存在。另外，这里假设network是reliable network，即可能有任意的delay，但是最终还是会将消息送达；reliable processors，即进程最终会受到消息并处理消息；one-shot transactions即一个事务请求一个sever只会一次。在这样的环境下吗，SNOW对于read-only transaction是不可能同时满足的，即No read-only transaction algorithm provides all of the SNOW properties。对于这样的一个场景，有Sa，Sb两个server，有一个client C-r发出一个读事务从Sa读取a，Sb读取b，另外一个client发出有冲突的写事务W更新a、b为新值。

* Lemma 1：The default behavior at servers returns values that are used by clients，因为要满足non-blocking 和 one response特性，就只能返回一次就使用；
* Lemma 2：Servers initially return old by default，因为读请求可能先到server，又不知道W，自然就会返回旧的值；
* Lemma 3. Servers eventually return new by default，paper中用反证法证明了server不会一直返回旧值；

由于2，3的存在，就会存有old value到new value的一个过渡的过程。在目前的系统假设下吗就一定肯存在一个R请求返回部分新部分旧值的情况。这样就违法了strict serializability。同样地，SNOW在Partial Synchrony网络下面也是成立的。

### 0x01 系统设计

根据SNOW Theorem，Paper中设计了两个系统，方便满足不同的特性。

#### COPS-SNOW

 原COPS系统不支持写事务，写一个一个可以的时候，会直接写一个数据中心内的server，然后被复制到其它的数据中心。这里会记录下这些写操作的causal dependencies，另外的数据中心在收到这些写请求的时候，会先检查其causal dependencies是否满足了。这样来满足causally consistent 特性。原来的 read-only transaction有causally consistent, non-blocking, two rounds几个特性，且not compatible with write transactions。COPS读取的时候先发送一个要读取的key list给server，server返回目前的values，和对于每个value的causal dependencies。第一个round client检查这里的consistent，即是否一个value对应的causal dependencies都得到了。如果不满足的话，还需要第二个round操作，请求没有满足dependencies的key的specific versions的值。这里将COPS优化为COPS-SNOW，为一个 latency-optimal read-only transaction的算法，其基本思路：

* COPS中，read-only transaction需要第二次的round来处理读只在一个部分看到了一个写的操作，而另外的一部分没有看到。原来的处理方式是第二次读取操作是保证没有看到的也能被看到。改进之后，如果一个read-only transaction操作的一部分没有看到其causal dependencies被观察到，那么对应的写入操作也不过应该被其它的部分看到。Client side的写入操作是一样的，read_only_txn基本上是一样的，不过不需要第二轮的操作。在server端，有几个关键的变动：1. server端读取的时候会检查old rdrs中是否存在相关的记录，这个old_rdrs在write的逻辑中添加的。如果有了则读取之前指定了时间戳的版本；2. 另外就是读取的时候会更新curr rdrs结构；对于写入来说，3. 会做一个dependency检查，检查写入值的causal dependencies是否被只读事务看到了。如果是的话更新old rds结构；4. 另外会将curr rdrs对于key的readers添加到old_rdrs，之后清空curr rdrs中的。

  ```
  // Client Side
  function read_only_txn(<keys>):
  	trans_id = generate_uuid()
    vals, deps = []
  	for k in keys # in parallel
  		vals[k], deps = read_txn(k, trans_id)
  	# update causal dependencies
  	return vals
  
  function write(key, val):
  	old_deps = get_deps()
  	new_dep = write(key, val, old_deps) 
  	# update causal dependencies
  	return
  
  // Server Side
  function read_txn(key, trans_id):
  	if trans_id in old_rdrs[key]
  		time = old_rdrs[key][trans_id] 
  		return val = read_at_time(key, time)
  	curr_rdrs[key].append(trans_id, logical_time.now())
  	return read(key)
  
  function write(key, val, deps): 
  	for d in deps # in parallel
  		old_rs = check_dep(d)
  		old_rdrs[key].append(old_rs) 
  	old_rdrs[key].append(curr_rdrs[key]) 
  	curr_rdrs[key].clear()
  	write(key, val)
  	# calculate causal dep for this write 
  	return new_dep
  
  function check_dep(dep)
  	# normal causal dependency check 
  	return old_rdrs[dep.key]
  ```

#### Rococo-SNOW

  Rococo-SNOW系统实现的是Strict Serializability，满足S，O，W三个，即这里的只读事务是可能被阻塞的。在Rococo原来的事务执行的逻辑中，一般分为coordinator指导来完成的三步，第一步为分发食物的操作到各个shards，然后检查直接有冲突的事务。第二步为所有的shards都知道了这些直接冲突的消息。第三步是可选的，来保证所有的shards知道了所有的transitively (but not directly) conflicting，即传递冲突。经过了这些操作之后，会对有冲突的顺序给出一个执行顺序，后面根据顺序执行。对于只读事务，特性是strictly serializable, blocking, multi-round, 以及 compatible with conflicting write transactions几个。第一轮操作中，只读事务会coordinator先发送请求到各个相关的shards，等待比这个事务先到的有冲突的事务执行。第二轮的操作方法和第一轮的相同，第二轮和第一轮的结果相同，则认为操作过程，如果不是则继续操作，

```
 ... This algorithm ensures strict serializability for the reads because it ensures they are totally ordered relative to all conflicting transactions. Waiting for all conflicting transactions to execute at a shard before returning in the first round ensures those transactions will have at least started at all other involved shards before the second-round read arrives. Thus, if a read-only transaction is not fully ordered before or after a write transaction, it will see different results and continue trying.
```

 Rococo-SNOW将这里只读事务的方法优化为optimal的，同样满足原来只读事务的特性。Rococo第二轮的操作主要是为了处理第一轮一部分看到了一个写入事务的操作而另外的没有看到。Rococo-SNOW避免这个是通过阻塞一个只读事务，直到有冲突的事务在对应的shard上执行了。这里由Rococo’s commit algorithm来保证每个事务操作的部分都能知道有冲突事务的transitive closure，即传递冲突相关的消息。如果在一个只读的一个shard操作没有观察到一个有冲突事务时候，这个shard就可以在冲突事务commit的时候让这个shard知道，这个shard可以知道对于只读事务应该返回那些数据。

### 0x02 评估

 这里的具体信息可以参看[1].


## Performance-Optimal Read-Only Transactions

### 0x10 基本内容

 这篇paper仿照前面的SNOW提出了一种NOCS Theorem，即N，O，C，S不能同时满足。与前面的SNOW Theorem一个差别就是W变成了C，这里的C指的是Constant metadata，即对于一个只读事务来说，其运行需要的metadata的数量是一个常量。而SHOW Theorem中的两个例子，都需要追踪相关的依赖、冲突关系，其运行需要的metadata不是常量。这里假设的环境和前面的SNOW是一样的。由于 network asynchrony的特性，这里总会存在一个unstable region，这里将这个unstable region定义为可能产生冲突，total order还没有完全建立的一个部分，而stable region为依据确定了的。证明：对于一个有两个server S1，S2的系统，存在多个的clients。这个系统中执行只读事务的算法满足N+O+S， R = {r1,r2}表示一个client C-R执行的一个只读事务的操作。另外的w1，w2为另外的一个client C-W发出的简单的写入请求。其中在real-time中，w1 happen before w2。对于这样的一个操作e1，

```
S1 : r1,w1 
S2 :      w2,r2
```

* Lemma 1. Without metadata, a read-only transaction that is N+O+S must observe any write that precedes it at a server。因为要满足N+O+S，必须要观察到前面的写入才行，负责就不能满足S or其它的条件了；
* Lemma 2. Processing e-1 while satisfying N+O+S requires dependency R → w1 to be transferred from S1 to S2。因为前面说了Lemma 1证明了必须要知道在其precede的写入操作，但是S2已经知道了，所以这里需要传输相关的信息。
* Lemma 3. Processing e-k while satisfying N+O+S requires Ω(k·M^2) metadata, for k = 1,...,N。即对于存在M^2+1个server的系统中，如果有N个clients，一般情况下这个N远大于M。满足N+O+S 的算法之上需要 Ω(k·M^2) 级别的元数据。这里在paper中有具体的证明。

NOCS同样可以用于指导系统的设计。对于满足NOC特点的系统，一般来说有这样几种方式：1. Explicit ordering control，这种方式可以显式地控制操作的顺序，比如读取操作可以读取stable region部分的数据，这样要求读取请求排序在写请求的前面；又比如一个是读请求的数据在unstable  region的时候，可以进行重新排序。由于排序的存在，这里不和strict serializability兼容的。2. Multi-versioning，读取的时候可以指定读取的版本。目前的也有一些基于MVCC的分布式事务系统，比如Spanner，MySQL Cluster，和Percolator等。这些读取操作不一定是optimal的，比如可能需要额外的一个round来获取timestamp，通过off-path messages的方式获取一个stable snapshots，或者是可能会阻塞。这里引入了一种称之为version clocks的方式来实现optimal的只读事务。

#### 0x11 系统设计

 这里的系统称之为PORT，其特性是Process-ordered serializability，这个特性保证了在一个进程观察到的legal total order。这个特性保留了有跨proccesces的total order之外的所有的trict serializability特性。一个client可能没有看到另外的一个client最近的更新，但是一定可以看到自己的。维护这个versiob clock的方式如下的伪代码。基本思路是获取the stable frontier来避免一些问题，这个stable frontier指的是最近的一个snapshot，这个snapshot中所有的写入操作都已经完成了。一个version clock追踪一个client连续的所有server的versionstamps，来获取这个versionstamp的最小的等一些信息。这个就作为这个client指导的 stable frontier。

* Clock structure，versionstamp保存在client本地，在每次读写请求的时候会带上这个信息。对于同样的versionstamp的同样类型的两个请求，server可以对其进行任意的排序。对于其不同类型的请求，将read排在write的后面。Server返回的时候，会返回器看到的最大的versionstamp。Client的view结构中记录了从server端来的max的versionstamp的信息，每次收到server的response的时候更新。这里这样对于一个client的读请求来说，其会有比前面的写请求更大的versionstamp，这样就能读取到自己写入的数据，保证read-your-writes特性。其递增的特性也保证了 process ordering。

```
// Client Side 
versionstamp = 0 # clock value
view[] # max known versionstamp per server


# Sending requests 
function get_vs_read():
	versionstamp = tick(min{view[]}) # stable frontier 
	return versionstamp

function get_vs_write(): 
	versionstamp++
	return versionstamp

# Receiving a response msg from server svr 
function recv_response(maxVS):
	view[svr] = max{view[svr], maxVS}
	if msg.for_write is true
		versionstamp = tick(maxVS)
	return

function tick(vs):
 	return max{vs, versionstamp}

// Server Side
maxVS = 0 # max seeen versionstamp
# ... return maxVS when sending response msg
```

 基本的Basic PORT设计中，数据保存在一个多版本的store中，而与server交互的client lib会维护version clock的信息。注意这里的逻辑只支持了单行写事务。Client在读取对于版本的时候，server端如果有对于的宝宝，则直接返回，如果没有，则查找更早的最佳的宝宝的数据，同时更新这里数据的max r vs的信息，来保证后面的写入请求的vs更大。这种操作在这里称之为Promotion。在接受到一个写请求的时候，这个请求到的vs比目前的max w vs更小的时候，这个写入请求就被忽略了，相当于被后面的写请求覆盖了。后面就是更加max r vs选择实际写入的宝宝，并更新VersionClock.maxVS的信息然后返回VersionClock.maxVS。这里Write omission在Paper中证明了会是安全的。

```
1 // Client Side
2 function read_only_txn(<keys>):
3    vs = VersionClock.get_vs_read()
4    for k in keys # in parallel
5 	    vals[k], maxVS = read(k, vs)
6 	    VersionClock.recv_response(maxVS) 
7    return vals # replies to end user 
8
9 function write(key, val):
10   vs = VersionClock.get_vs_write()
11   maxVS = write(key, val, vs)
12   VersionClock.recv_response(maxVS)
13   return # replies to end user 
14
15 // Server Side
16 vers[keys][] # multi-versioned storage
17 function read(key, vs):
18    if vers[key][vs] exists
19 	      return vers[key][vs], VersionClock.maxVS
20    else # return nearest version to not block
21        near_vs = find_nearest_earlier(ver)
22        # ensure future writes have higher vs
23        vers[key].max_r_vs = max(vers[key].max_r_vs, vs)
24        return vers[key][near_vs], VersionClock.maxVS
25
26 function write(key, val, vs):
27    if vs <= vers[key].max_w_vs
28       return VersionClock.maxVS # omit write
29    if vers[key].max_r_vs >= vs
30       vs = max_r_vs + 1 # commit after promoted versions 
31    vers[key][vs] = val
32    vers[key].max_w_vs = vs
33    if vs > VersionClock.maxVS
34       VersionClock.maxVS = vs
35    return VersionClock.maxVS
```

在这样的执行模式下面，只读事务是有可能读取到staleness的数据的。这里通过多种方式来减少data staleness的情况：比如通过主动追踪stable frontier；比如通过co-locate client和server的方式。除此之外，这里还对Eiger进行了改进，来实现前面系统同样的特性。在Eiger中，promotion策略不行是因为其不能保证一个写入事务中，所有的写入操作都在在所有的servers上同时被 promoted。这里就采用了一种per-client ordering的策略，一个client可以获取任意client最近的stable frontier之外的写入操作信息，这样client就即获取到 stable frontier，有可以看到自己的写入。

* Eiger-PORT中，每个client维护了一个lst map，记录下每个server的local safe time，lst。另外一个gst表示global safe time，servers中lst最小的。Gst这里会在只读事务中当做read timestamp使用。这个gst在请求一个key的时候会带上，除此之外还会带上这个client的唯一ID。Server返回的时候会带上lst。Client发出一个写事务的时候，一个server被随机选为coordinator。写入请求会带上key、value和client ID以及gst的信息。

* Server在处理写入的时候，先记录下目前的lamport time，然后创建一个pending version。一个pending wtxns会记录下进行中的写事务。Pending writes其时间戳不会小于lst。2PC第一步接受之后，返回的信息里面带有 prepared time信息。这里会保证pending time小于这个prepared time。在commit的时候，计算出prepared time中最大的值，然后coordinator发送commit请求。其它的sever在接受到这个请求之后，提交local pending version，版本为commit time。然后更新pending wtxns和lst的信息。

* Server在处理写入的时候，获取read timestamp，rts对应的版本。如果同一个client进行了一个更晚的写入，则返回这个写入的数据。如果获取的版本为同一个client写入的，则为了保证write isolation，需要检查在这个版本的gst和commit time中有其他版本的存在。gst为发出请求的时候带上，二commit time为提交的时候赋值，这里就是检查这个过程中的写入情况。如果存在但是是其它client写入的，则直接返回。负责要递归查找下去，如下伪代码，

  ```
  function find_isolated(ver):
      # iterate from newer version to old 
      while v in DS[k].newer_than(ver.gst)
          and v in DS[k].older_than(ver.commit_t)
          if v.cl_id != ver.cl_id
             return v 
          else
             return find_isolated(v) 
       return ver
  ```

### 0x12 评估

 这里的具体信息可以参看[1].

## 参考

1. The SNOW Theorem and Latency-Optimal Read-Only Transactions, OSDI '16.
2. Performance-Optimal Read-Only Transactions, OSDI '20.