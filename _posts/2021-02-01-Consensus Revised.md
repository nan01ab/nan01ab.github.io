---
layout: page
title: Distributed Consensus Revised
tags: [Consensus, Distribution]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## Distributed Consensus Revised

### 0x00 引言

 这篇《Distributed Consensus Revised》是一篇博士论文，因为某个大佬的点赞以此被更多人知晓。其讨论的还是Distributed Consensus算法的一些优化设计的讨论。在paper的前面，讨论了一个简单的Signal Acceptor，即只有一个Acceptor的一个Consensus算法，和Classic Paxos的一些特性。其总结了Classic Paxos这样的几点Properties，这里的epoch应该是一般的proposal id的含义：

```
Property 1. Proposers use unique epochs for each proposal. 
		即每个proposal会有独特的epcoh。
Property 2. Proposers only propose a value after receiving promises from ⌊n/2⌋ + 1 acceptors.
		收到了大部分的acceptors的回复，这样就能保证之前有更大提案号proposal会被这次知道，以及chosen的value也会被这次操作知道。
Property 3. Proposers only return a value after receiving accepts from ⌊n/2⌋ + 1 acceptors.
    即认为一个value被chosen了只有在多数acceptors接受之后，这样能保证一个value一旦被chosen就不会chosen其它的value。
Property 4. Proposers must choose a value to propose according to the value selection rules. If no previously accepted proposals were returned with promises then any value can be chosen. If one or more previously accepted proposals were returned then the value associated with the highest epoch is chosen.
		即之前没有chosen的value的时候，总是可以任意决定这次的value，而如果之前有的会需要选择之前以及被chosen了的epoch最大的那个对应的value。保证了chosen的不会被覆盖。
Property 5. Each epoch used by a proposer is greater than all previous epochs used by the proposer.
		即选择epoch的时候选择更大的，这么保证算法能够往前推进。
Property 6. For each prepare or propose message received by an acceptor, the message is processed by the acceptor only if epoch received is greater than or equal to the last promised epoch.
		即不会处理比已知的epoch更小epoch的操作。这里有一个细节就是这个 >= 已知最大的epoch，而不是 >.
Property 7. For each prepare or propose message received, the acceptor’s last promised epoch is set to the epoch received. This is after Property 6 has been satisfied.
		即总是会选择看到的最大的epoch的proposal。
Property 8. For each prepare message received, the acceptor replies with promise. This is after Properties 6 & 7 have been satisfied.
	  只要满足了Properties 6 & 7，prepare message会被处理然后回复消息，避免不处理造成使算法无法推进。
Property 9. For each propose message received, the acceptor replies with accept after updating its last accepted proposal. This is after Properties 6 & 7 have been satisfied.
		即更新了last accepted proposal之后在回复，避免了出现丢失更新/违反自己的promised。
Property 10. Last promised epoch and last accepted proposal are persistent and only updated by Properties 7 & 9
    即数据需要持久化，避免丢失已经promised的消息，而导致可能违反自己的promise。
```

在这些Properties上，paper对其Safety、Progress等特性进行证明 or 讨论，Paxos这样的Distributed Consensus算法不能满足termination的特性。这个是FLP以及证明了的。

### 0x01 Known Revisions 

Known Revisions这一章总结了在操作的流程上面的一些优化/改动。1. 比如Negative responses (NACKs)，在Classic Paxos算法中，对于接受到了比已知最大epoch更小的请求，会直接忽略处理。这样会使得发送这个请求的proposer超时。这里一个直观的优化就是这个proposl不能被接受的时候返回一个拒绝的信息，告诉proposer做另外的动作，比如重试等；2. Bypassing phase two策略的优化思路则是phase的时候得到回复里面就是有半数以上的acceptors 接受了同样的proposal，后面这样proposal都不会成功 or 会选择这个已经chosen了的值。所以这种情况下可以直接在phase 1完成之后就返回。可以想象这种情况一般出现在之前的proposer提出的value被chosen了，但是其在接受到被半数以上value被chosen的回复之前 or 知道chosen了还没来得及进行其它操作就挂了。这样的话出现的概率应该不大；3. 另外一个优化是关于Termination，Classic Paxos算法一个acceptor只能在和proposer交互之后才能知道被chosen了的值。这里的优化方式值加入一个可选的phase 3，一个proposer知道一个value被chosen之后，可以发送一个decided(v)消息给所有的acceptors，让其知晓情况。这样分别实现前面第2点的优化。除了这些的另外一些优化：

* Distinguished proposer，这点改进了两个/多个proposers恶性竞争导致的算法无法完成的问题。可以选择一个proposer作为distinguished proposer，其它的non-distinguished proposer将请求转发给这个distinguished proposer。如果一个non-distinguished proposer认为其distinguished proposer故障的情况下可以直接自己提出proposal。这里只是降低了活锁的概率，并不会完全拒绝也不能解决这个问题。

* Phase ordering，Multi-Paxos，Roles.......

* Epochs，这个优化点事降低持久化epoch信息的开销，可以每次申请一段的epochs，持久化这个epochs的最大值。是一种常见的批量申请的方式。

* Phase one voting for epochs，这里主要基于观察到Classic Paxos不要去epoch的唯一性。Classic Paxos是通过对于一个给定的epoch，只有一个proposer进入phase 2，这样就使得能完成第一步的proposal的epoch是唯一的。这里的优化是提出通过voting的方式来达成epoch的唯一性，基本的方式是在接受到prepare(e)消息的时候，加入一个(e=e-pro ∧p=plst)的判断，即如果这次提出的proposer为上次promised的prepare消息的同一个，其也可以完成prepare阶段。这里相对于发送了对epoch的一些要求。

  ```
  state :
  	• plst: last proposer promised to, plst ∈ P (persistent)
  while true do 
     switch do
  	case prepare(e) received from proposer p
  	  if (e-pro =nil)∨(e>e-pro)∨(e=e-pro ∧p=plst)then
           e-pro ←e,plst ←p
           send promise(e,e-acc,v-acc) to proposer 
      case propose(e,v) received from proposer
        if e-pro = nil ∨ e ≥ e-pro then
          e-pro ← e
          v-acc ←v,e-acc ←e
          send accept(e) to proposer
  ```

* Proposal copying，这个改动基于这样的一个观察：当一个acceptor接受到一个propose(e,v)消息的时候，其可以知道proposer已经通过epoch e完成了phase 1，另外一个后面的选择value的时候会关联到这个epoch。这里其它其它的proposer使用这个propose(e,v)中的(e, v)也没关系，从而跳过第一步。这里将其称之为proposal copying。这个改动可以用于实现efficient recovery from NACKs，基本思路是NACKs中带上最好一个accepted的proposal (g,w)。Proposer接受到一个带proposal (g,w)的NACKs的消息的时候，可以使用proposal (g, w)跳过phrase 1。不过这个优化感觉没啥用，还会导致其它的问题。

* Miscellaneous，这里是一些杂项的优化。比如1. Learning，即加入一个learner的角色，其可以通过proposer or acceptors来学习到被chosen的value，不参与choose value的逻辑。这个在很多复制一份数据到其它地方有用；2. Messages，比如避免向所有的acceptor发送消息；3. Stricter epoch conditions，Classic Paxos对消息接受的要求是其epoch大于等于最后一个promised的epoch，更严格的要求是promise要求e > e-pro；4. Fail-stop model，可以要求acceptor在故障之后不能直接重新参与来方便实现不用持久化promise，这样需要reconfiguration操作来满足集群中节点的数据是足够的； 5. Virtual sequences，即一种批量处理的思路；6. Fast Paxos，即Fast Paxos。

### 0x02 Quorum Intersection Revised

这里优化的思路和Flexible Paxos的思路差不多，用集合的方式描述就是Classic Paxos对每一步操作中对quorum的要求是
$$
\\ ∀Q,Q′∈Q1 : Q∩Q' \neq ∅
\\ ∀Q,Q′ ∈Q2 :Q∩Q′ \neq ∅
\\ ∀Q1 ∈Q1,∀Q2 ∈Q2 :Q1 ∩Q2 \neq ∅,
$$
即phase 1操作的时候，不同作出对不同prepare消息promise的quorum必须有交集，同理phase 2也是必须有交集，而且同一个choose操作的第一步和第二步参与的quorum中也必须有交集。而这里的改动是是需要满足第三个条件即可。在这个的基础之上，quorum intersection across epochs这个的思路是在quorum与epoch联系起来。使用$Q_n^e$表示第n阶段epoch的quorum set。则上面的条件表示为
$$
\\ ∀Q∈Q^e_1,∀f ∈E,∀Q′ ∈Q^f_2 :Q∩Q′ \neq ∅.
$$
然后继续优化放松这个Intersection的要求：发送为要求一个epoch e对应的phase 1的操作的quorum，和比e小的epoch对应的quorums有交集即可。这里发送了同一个epoch 第1、2阶段的交集的要求。对应e更大的epoch的phase 2操作的quorum也就不要求和e对应的phase 1的操作的quorum相交了。表示为：
$$
\\ ∀Q ∈ Q^e_1, ∀f ∈ E : f < e ⇒ ∀Q′ ∈ Q^f_2 : Q ∩ Q′ \neq ∅.
$$
Co-location of proposers and acceptors的思路如果proposer和acceptor是同一个进程的话(一般实现的思路都是将其作为一个进程)，这里的几个优化有点意思：

* All aboard Paxos，这个的思路是解释以有3个节点的系统为例，如果将phase 1和phase 2可能的quorum几个划为
  $$
  \\ Q1 =\{\{a1\},\{a2\},\{a3\}\}
  \\ Q2 =\{\{a1,a2,a3\}\}
  $$
  也就是说phase 1操作的时候只需要在本进程处理，而phase向所有的进程发送消息。这样就能保证同一个前面要求的集合相交的要求了。这样可以实现在one round trip就实现consensus。不过这个的一个限制是所有的节点都必须存活，否则就完成不了第二步了。为了处理这个问题，其优化思路是：对于0 到 k的epoch，要求所有的节点都在phase 2都accepted，对于k+1的epoch只要求半数以上的accept，而对于k+2以上的，则要求一般以上的promised和accepted，表示如下图。这里的如何选择k的话，应该就是在前面的all accepted添加无法满足的时候，比如在k+1的时候发现无法满足all accepted的时候，在接下来的epoch退回到一般的方式。

  <img src="/assets/png/consensus-revised-abroad.png" style="zoom:80%;" />

  Paper中的内容值讨论到了上面的内容。如果这样的话，只要有一个消息无法达成all accepted就会导致其退化，而无法使用优化。可以继续的优化策略：这里可以选择的，1. 一种方式是在发现在一个epoch n前面的都达成了一致的话，有可尝试恢复到使用优化的方式上；2. 另外一个是目前的node set无法达成all accepted的时候，可以切换node set。

* Singleton Paxos，quorum组织的方式和All aboard Paxos相反，这里是phase所有的节点参与，即Q1 =\{\{a1,a2,a3\}\}，Q2 =\{\{a1\},\{a2\},\{a3\}\}。

* Majority quorums for co-location，思路是using different quorums for different epochs。

* Multi-Paxos，Multi-Paxos在quorum方面的一些讨论，比如同时向多少acceptor发送messages。

### 0x03 Promises Revised & Value Selection Revised

Classic Paxos从phase 1推进到phase 2的条件是在phase 1收到了半数以上acceptors的promises，而与这个primise的内容是没有关系的。前面Quorum Intersection Revised的改进是在phase 1能够推进的条件是和前面的进行phase 2的epoch的acceptor之间有交集。要求有这个交集的原因是proposer不清楚在phase 2那个quorum被使用。而如果一个proposer在一个epoch e的操作，在phase 1收到一个promise(e,f,v)(e: epoch, f: last accepted epoch, v: last accepted value, f and v mayne nil)，这个proposer可以从其中知道在epoch f的时候其选择的值为v。此时proposer不需要等待来自 f 在phase 2的quorums的promises，这个时候quorum set相交的要求可以放松，因为已经知道f及其之前的选择的为v：
$$
\\ ∀Q ∈ Q^e_1,∀g ∈ E: f< g <e ⇒ ∀Q′ ∈ Q^g_2 : Q ∩ Q′ \neq ∅.
$$
即对于任意在f和e之间的epoch，要求e操作phase 1的quorums和g操作的phase 2的quorums有相交。称之为revision B quorum intersection requirement，而这个称之为revision C quorum intersection requirement。开始的就phase 1和phase 2有相交要求的称之为revision A quorum intersection requirement。Paxos phase1的操作有两层的意思：第一个是了解之前是否之前已经选择了一个value，第二个是防止一个value在这个操作和后面的操作过程中选择了一个value，

```
 If a proposer in e receives promise(e,f,v) then it learns that all epochs ≤ f are limited to value v. This is because the proposer of f must have received promises from a quorum of acceptors in its phase one. The proposer of e can essentially reuse the phase one that was successfully executed by the proposer in f as the outcome of phase one in epoch f is known to be value v.
```

另外一个改进的是Value Selection的算法，Classic Paxos选择value的算法是在phase 1收到了带value的promise消息中，则选择其中epoch最大的，而如果是任何的value则自己决定一个value。这里的改动是增加了一个map，记录一个accepts a到一个(f, w)元组的映射，没有的话元组为no/nil。这个(f, w)来自phase 1接收到的消息promise(e,f,w) 。原来的value selection算法选择出来的可能的value集合为：
$$
\\ \{v ∈ V | ∃f ∈ E : R[\_] = (f, v) ∧ (∀a ∈ A: R[a]=no ∨ ∃g ∈ E:R[a]=(g, \_)∧f≥g)
$$
其中R为就是这个映射关系。其满足： for each acceptor a ∈ A, either: – no: no promise received yet from a；– (e, v): the proposal received with a promise from a, maybe nil。这个选择的方式其意思是: possibleValues的集合是对于epoch f有某个acceptor选择了的value 且满足 对于所有的acceptor大于这个f的epoch要么没有选择的value，要么是选择了value的epoch不大于这个f。用大白话说就是要买是空，要么是有最大epoch相关的value。而这里给出的是一个Quorum-based。其算法表示如下：算法先通过一个循环中的三个判断来发现是否有quorum选择了一个value。然后返回这个D中对应的value的集合。这判断条件主要基于：

```
* If an acceptor a sends promise(f,e,w) where (e, w) = nil then no decision is reached in epochs up to f (exclusive) by the quorums containing a
Lemma 20. If acceptors a1 and a2 send promise(g,e,w) and promise(g,f,x) (respectively) where e < f and w != x then no decision is reached in epochs up to g (exclusive) by the quorums containing a1.
```

<img src="/assets/png/distributed-consensus-possible-values.png" style="zoom:80%;" />

这里的quorum是和epoch无关的，前面的revision B 和 C中quorum和epoch相关，paper有讨论了在revision B 和 C中和epoch联系的Epoch dependent 算法。 Promises Revised & Value Selection Revised这两个部分主要还是在quorum上面做文章，这里的方法会导致算法变得复杂了很多。

### 0x04 Epochs Revised 

 这部分讨论的是epoch的优化，给出了3个对于epoch的改进思路：

* Epochs from an allocator，这部分的思路是将epoch的分配交给一个allocator，将epoch分为两个部分sid: sequence number和vid: service version number (persistent, initially 0)。持久化只需要持久化vid，生产的时候将sid+1拼接上vid即为新的epoch。vid在每次启动的时候增加，paper中些的是sid←0,vid←vid+1，不过应该是增加了就是可以的。

* Epochs by value mapping，epoch用于区分不同的proposal。如果有一种方式将一个value映射到一个epoch，则这样就不需要持久化epoch，也不需要phase one quorum intersection的要求。这里带来了一些其它的限制：比如需要先知道propose的value，以及提出的value可取的值是有限的集合。paper中距离对一个Binary值达成一致的例子，value取值只能是0 or 1，奇数的epoch对应到value = 1，而偶数的对应到value为1。达成一致最好情况下可以减少到一次操作。

* Epochs by recovery，这里讨论的是如何移除value uniqueness的requirement。之前的方法是要求一个epoch对应唯一的一个value。简单的移除这个限制会导致其它的一些问题，paper中讨论了这些问题并给出其解决方法：1. 因为phase 2操作的不相交的quorums接收了不同的值，造成不同的proposers提交了不同的值，这里就要求phase 2操作的quorum要想叫。即$∀Q,Q′ ∈Q^e_2 :Q∩Q′ \neq ∅$；2. phase2操作可能覆盖以及选择了的value，这里的处理方式是对accept过epoch，要求其value也和之前accept的相同。3: 如果收到两个value要求choose/accept(第二阶段的操作)，则会有value collision。这里的解决方式是增加不同操作quorums交集的要求，表示如下。即对epoch f其phase 2操作的两个quorums，要求后面epoch e的phase 1的操作的quorum之前存在交集。
  $$
  \\ ∀Q∈Q^e_1,∀f∈E:f<e ⇒ ∀Q′,Q′′ ∈Q^f_2 :Q∩Q′∩Q′′ \neq ∅. \\
  $$
  感觉这样做的目的是什么呢？

Paper中还有一些对epoch改动的讨论，以及整体的Conclusion。

### 0x05  总结

 这里可以参看[1].

## 参考

1. Distributed Consensus Revised，*Technical Report*，https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-935.pdf

