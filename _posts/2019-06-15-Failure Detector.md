---
layout: page
title: Failure Detector and Membership Protocol
tags: [Distributed]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## The φ Accrual Failure Detector

### 0x00 引言

   这篇Paper提出的算法被后面不少的分布式系统使用。一般故障探测的思路是给出一个节点(or 进程)是否已经故障的bool型的判断。而这篇Paper的思路则从故障探测不是完全可靠的出发，给出的是一个是否故障的概率值。

### 0x01 基本概念

 故障探测中，实际的环境中时很难区分真正的故障还是low process。有这样的两个性质，

```
Property 1 (Strong completeness) There is a time after which every process that crashes is permanently suspected by all correct processes.

Property 2 (Eventual strong accuracy) There is a time after which correct processes are not suspected by any correct process.
```

  process故障与否的情况，被别的进程认识到从市存在一个时间差。由于存在误判和从故障到意识到故障的时间查，评价一个故障探测器的两个重要指标一个市故障发现的时间，另外一个就是误判的概率,

```
Definition1(Detection time T_D) The detection time is the time that elapses since the crash of p and until q begins to suspect p permanently.

Definition 2 (Average mistake rate λ_M ) This provides a measure of the rate at which a failure detector generates wrong suspicions.
```

现在的很多故障探测时基于心跳的方式，假设一个进程q监控另外一个进程p的健康情况，

* Heartbeat failure detectors，最常见的方式就是周期性地p向q发送心跳消息，如果在一个预定义的时间段∆to内，q没有收到q的心态，q就认为p已经发生故障。∆to大小的设计决定了故障发现的时间和误判的概念。Heartbeat failure detectors这里可以采用的一个优化方式就是基于transmission time来决定一个∆to。
* Adaptive failure detectors，比如Chen-FD方法的思路是基于网络流量的概率性分析，这个协议使用最近几次的到底时间去样来预测下一个心条达到的时间。∆to的设置就根据这个预测的时间在假设一个固定的safety margin α。Bertier-FD方法的基本思路和Chen-FD是一样，不过这个safety margin α不是一个固定的值，而是根据RTT计算出来的一个值。

### 0x02 基本思路和实现

  The φ Accrual Failure Detector的基本思路是使用是否故障的概率判断代替是否故障的bool类型的判断。这里使用 susp_level_p 来表示评估进程p是否已经故障的函数，susp_level_p(t)的输出会有以下的性质，

```
Property 3 (Accruement) If process p is faulty, the suspicion level susp_level_p(t) tends to infinity as time goes to infinity.
  即如果进程确实故障，susp_level_p输出的值会随着时间趋向于无穷大.
  
Property 4 (Unknown upper bound) If process p is correct, then susp_level_p(t) is bounded.
  如果进程p是健康的，则函数输出值是有界的.
  
Property 5 (Reset) If process p is correct, then for any time t0, susp_level_p(t) = 0 for some time t ≥ t0.
  如果进程p是健康的，则在一个时间短后，这个susp_level_p输出为0.
```

 在实现中，The φ Accrual Failure Detector使用基于Window Size的心跳时间采样的方法，使用一个固定长度的队列记录最近若干次的心跳间隔时间，使用以下公式计算φ(T_now)，
$$
φ(T_{now}) = -\log_{10}(P_{later}(T_{now}-T_{last}))
$$
其中T_last表示最近一次收到心跳的时间，T_now表示这次收到心跳的时间。而P_later的定义如下，
$$
P_{later}(t) = \frac{1}{\sigma\sqrt{2\pi}}\int_t^{+\infty}e^{-\frac{(x-u)^2}{2\sigma^2}}dx = 1 - F(t)
$$
后面是一个正态分布的积分。这里主要要处理的参数就是期望和方差。这两个值就使用之前Window Size采样计算出来。The φ Accrual Failure Detector整体的思路如下图。从上面的方程来看，T_last - T_now会被函数P_later转化为0 - 1 之间的一个值，-log10的函数在(0, 1)可去值为0到+∞，如果t越多，P_later积分得出的值就会约小，而会使得φ(T_now)得出的值越大。

![phi-detector-arch](/assets/images/phi-detector-arch.png)

### 0x03 评估

这里的具体消息可以参看[1],

## SWIM: Scalable Weakly-consistent Infection-style Process Group Membership Protocol

### 0x10 引言

在分布式系统中，通常是一个节点的集合一起来实现一个系统。而Membership Protocol就是处理这个集合中的节点如果认识这个集合中另外节点的Protocol。SWIM节点主要要处理两个问题，一个就是节点的故障探测，另外一个就是信息的传播。从SWIM名字上就可以看出SWIM成员协议的几个特点：1. Scalable，可拓展，2. Weakly-consistent 弱一致性，3. Infection-style，类似Gossip一样的与集群中部分节点交换信息的方式，而不是全部广播。

### 0x11 基本思路

SWIN包含两个主要的组件，1. Failure Detector Component，用于探测成员的健康状况，2. Dissemination Component，用于发送成员状况信息的组件。

* SWIM Failure Detector，SWIM的故障探测器使用2个参数，一个是一个协议的周期T'，和一个整数K。每一个周期内，集群中的一个节点随机选择一个节点，像其发送ping消息，并期待它恢复ack。如果在一个时间内没有收到这个选择节点的ack回复。这个时候并不会直接认为这个节点已经挂掉了，而是会选择另外的K个节点，向它们发送ping-req消息。在收到这个ping-req消息之后，节点会使用ping消息来探测指定节点是否存活。这种方式避免来一个节点做出决定，这样更加容易发生误判。如果在这个时间周期内，主动探测的节点没有收到直接的ack回复，也没有收到K个节点间接的ack回复，这个节点在自己的local membership list中标记这个被探测的节点为故障。之后被Dissemination Component处理。

* Dissemination Component，节点失败的消息使用多播的方式发送给集群中的其它节点。对于主动离开和新节点加入的情况也可以使用同样的方式。新加入的节点至少要知道集群中一个已经存在的节点才可以加入。也可以使用一个指定的节点来处理新节点的加入，

  ```
  This can be realized through one of several means: if the group is associated with a well known server or IP multicast address, all joins could be directed to the associated address. In the absence of such infrastructure , join messages could be broadcast, and group members hearing it can probabilistically decide (by tossing a coin) whether to reply to it. 
  ```

### 0x12 优化

  SWIM的基本协议如上。除此之外，Paper中还提出了几项优化的措施，

* Infection-Style Dissemination Component，在最初的设计中，Dissemination Component使用的是多播的方式，但是这个在很多的情况下是不能使用的。如果不能使用就只能使用一个个的“广播”的形式，比较低效。Infection-Style Dissemination Component就是为了优化这个问题。这里优化的一个讨巧的设计就是在故障探测组件离开的ping、ping-req和ack消息中捎带上要广播的消息。
* Suspicion Mechanism，主要目的是用于减少误判的概率。这个其实不能说是一个优化，而是一个在故障发现时间和误判概率之间的一个权衡。在一个节点探测的周期中，如果它没有直接也没有间接收到被探测节点的ack消息，不是将其标记为故障而是标记为怀疑故障。怀疑节点会被暂时当作是非故障节点，如果其它节点能够收到被探测节点的ack，则发送消息“纠正”这个怀疑，被探测节点本身收到自己被怀疑的消息也是要做“纠正”处理。如果在一定时间内没有收到这个被探测节点依然存活的消息，那这个节点就被标记为故障。
* Round-Robin Probe Target Selection。简而言之就是不是使用随机选择的方式，而是造member list里面依次选择。

### 0x14 评估

  这里的具体信息可以参看[2].

## 参考

1. The φ Accrual Failure Detector, SRDS ’04.
2. SWIM: Scalable Weakly-consistent Infection-style Process Group Membership Protocol, DSN '02.

