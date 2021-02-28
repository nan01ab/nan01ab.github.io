---
layout: page
title: IO Scheduling
tags: [Storage, Operating System]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## mClock: Handling Throughput Variability for Hypervisor IO Scheduling

### 0x00 基本思路

  mClock是一个结点的IO QoS算法，这篇Paper描述的是在Hypervisor上面运行多个VMs的情况下如何调度VMs之间的IO请求，当然mClock算法完全不仅仅可以用到这个Hypervisor  IO Scheduling方面，是有不同的IO请求需要控制其共享存储设备能力的都可以用到mClock。mClock算法的核心是使用三个值来控制不同请求原来的QoS，shares (即 weights)即表示一个请求来源的权重，reservations表示至少为一个请求来源预留多少资源，表示一个下限，而另外一个limits值表示能够使用资源的上限。一般情况下，为一个请求来源分配的资源为weights值在总的值占的比例，也就是说mClock是一种Proportional Share Algorithms。而reservations和limits的加入是处理资源争用的情况，即在根据weight分配的资源不足reservation的值时候，会增加到reservations的值，而根据weight分配的资源超过limit的时候，会减少到limit的。后面就是如何实现。

### 0x01 算法

mClock实现的时候，使用为每个请求打tag的方式。打tag的方式沿用了mClock之前算法的一些方式。以权重为例，其基本思路是根据权重w，每次间隔1/w-i的间隔给一个来源的请求打上tag。处理IO请求的之前，根据这个tag值排序，然后处理这个请求，这样就能实现按照权重共享资源的效果。使用R，L，P分别表示Reservation，Limit和Share-Based-Tag(Weight)的tag，使用w，r，l分配表示权重，Reservation和Limit值。则其打不通tag的方式是一样的。都是按照如下的方式：
$$
\\ R_i^r = \max{\{R_{i}^{r-1}+ 1/r_i, CurrentTime\}},
\\ L_i^r = \max{\{L_{i}^{r-1}+ 1/l_i, CurrentTime\}},
\\ P_i^r = \max{\{P_{i}^{r-1}+ 1/w_i, CurrentTime\}}, \\
$$
也就是谁其打tag的基本算法是一样的。即理想的情况下，请求被安排到1/r or 1/l or 1/p间隔的时间点被处理，从而实现满足R，L和P的条件。但是一段时间内，分配给一个请求来源的没有使用分配的资源的时候，其也不会后面补偿，使用取下一个间隔时间和目前时间的最大值。在此的基础之上，由于w是一个相对的值，想要进行特殊的一下处理。R和L的标准是固定的，所以不用特殊处理。P表情的原因是新client or 一个来源重新变得活跃的时候，新来请求会其打tag的基准点不一样了。可能导致饥饿问题的出现(一个请求来源的P tag可能已经超出了目前时间，新来的以目前的时间为基准)：

```
 The initial P tag value of a freshly active VM is set to the current time, but the spacing of P tags after that is determined by the relative weights of the VMs. After the VM has been active for some time, the P tag values become unrelated to real time. This can lead to starvation when a new VM becomes active, since the existing P tags are unrelated to the P tag of the new VM. 
```

为了处理这个问题，mClock使用了调整基准点的方法。即请求来源的P减去minPtag - CurrentTime t。在打tag知乎，就是处理请求。调度请求的时候要考虑3个不同的tag。基本的算法表示如下，核心的部分是分为两个部分：优先处理reservation的请求，因为要保证每个来源的reservation值，这里通过检查其R tag和目前时间t的比较。优先处理reservation的请求之后，处理工具weight来分配资源的请求(weight-based phase)。处理一个请求之前，要先过滤掉那些已经超过limit的请求，根据L tag与目前时间t来判断。然后根据P tag，选择最新的P tag来处理。处理P的时候，也会满足reservation值的要求，这样要在reservation中减去一个给出的资源。方式是对于每个R tag减去 1/r-k，其中r-k为第k个请求来源的r值。

<img src="/assets/png/mlcok-alg.png" style="zoom:80%;" />

 上面是一般情况下处理的逻辑，Paper中还提到了一些特殊情况的处理，如IO bursts, request types, IO size, locality of requests 以及reservation settings等。Burst Handling，这个优化值针对请求有比较大的spatial locality的情况下。对于处于idle状态一定之后，给一个idle credits达到一个类似补偿的效果。基本思路是max选择的时候第二个参数不再是CurrentTime，而是其减去一个值σ。表示为，
$$
\\ P_i^r = \max{\{P_i^{r-1}+1/w_i, t - σ_i/w_i\}}
$$
Request Type，另外一个是请求类型的考虑，比如读和写的区别。mClock实际上不区分这个；IO Size，不同的IO Size的请求开销不同，可以根据比较固定的一些开销加上根据IO Size变化的开销，就算出一个比例值，一个大请求看作是多个一般的请求。

#### Distributed mClock

mClock的分布式版本称之为dmClock。这种请求下，IO请求会发送到多个的存储server。从算法上，dmClock在mClock上改动不大，不过应该实现上会麻烦很多。dmClock中打tag要考虑两个东西，δ和p。其中δ-i表示一个请求来源v-i，从上次v-i来的请求这个服务器，从目前发送这个请求的时候(即有发送给这个服务器的时候)，请求这个服务器之外的其它服务器完成的请求，加上目前的这个请求的个数。类似，p-i表示内容是一样的，只是只计算Reservation类型的请求。由此打tag的方式变为如下公式，如果δ和p为1的话，即为mClock。
$$
\\ R_i^r = \max{\{R_{i}^{r-1}+ p_i/r_i, CurrentTime\}},
\\ L_i^r = \max{\{L_{i}^{r-1}+ δ_i/l_i, CurrentTime\}},
\\ P_i^r = \max{\{P_{i}^{r-1}+ δ_i/w_i, CurrentTime\}}, \\
$$
这里增加δ和p，实际上就是增加一种delay，因为会是的打的tag更可能变大，即变得更靠后。δ和p也类似于给r、l和w增加了一个倍数。而这deplay的值 or 增加的倍数来自于一个client对同一个server的两个请求之间的发送给其它的server请求的数量。这样，每台server在运行给自的调度算法的时候，不需要知道其它的server接受到请求的状态。

### 0x02 评估

 这里的具体信息可以参考[1].

## 参考

1. mClock: Handling Throughput Variability for Hypervisor IO Scheduling,  OSDI '10.

