---
layout: page
title: Fine-grained TCP Retransmissions
tags: [Transport Protocol, Data Center, Network]
excerpt_separator: <!--more-->
typora-root-url: ../
---



## Safe and Effective Fine-grained TCP Retransmissions for Datacenter Communication 



### 引言

  TCP中对于超时重传的时间处理上，为了避免虚假的超时重传，设置来一个最小的超时时间。在Linux上，这个是时间一般是200ms。在广域网上，这个最低的值设置是有意义的。但是在延时很低的数据中心内部，设个时间限制就可能导致一些问题，特别是incast的情况(就是多台机器与一台机器同时通信)。

  这篇Paper就讨论了减小这个RTO带来的影响，发现可以通过将这个RTO的最小值设置的很小也能保证安全和效率，同时解决存在的问题。



### 问题分析

 现在常见的RTO计算方式是:

```
RTO = SRTT + (4 × RTTVAR)
```

考虑到指数退让，这里就是:

```
timeout = RTO × 2 ^ backoff
```

.

模拟中这个RTO的最小值对性能的影响，可以看出来RTO的最小值对性能有着明显的影响(模拟情况):

![fine-rto-simulation](/assets/img/fine-rto-simulation.png)

​	Paper中的在实际情况中的测试，也显示出减小RTOmin对性能的影响。

```
Despite these differences, the real world results show the need to reduce the RTO to at least 1ms to avoid throughput degradation at scales of up to 16 servers.
```

  此外，除了减小RTO能带来明显的好处外，给timeout一个随机的元素也能带来良好的效果:

```
timeout = (RTO + (rand(0.5) × RTO)) × 2 ^ backoff
```

模拟结果：

![rto-fine-rand](/assets/img/rto-fine-rand.png)



### 实现

  实现上主要修改的地方就是利用linux的高精度时间子系统，解决之前TCP stack中时间测量过于粗糙的问题。

```
With the TCP timestamp option enabled, RTT estimates are calculated based on the difference between the timestamp option in an earlier packet and the corresponding ACK. We convert the time from nanoseconds to microseconds and store the value in the TCP timestamp option.2 This change can be accomplished entirely on the sender—receivers already echo back the value in the TCP timestamp option.
```

.

>

### 安全否?

  其实上面减小RTOmin能带来的效果是很显然的，主要是安全问题？是否会导致大量的假的超时，从而导致重传反而降低了性能呢？Paper的观点当然是很安全啦。

  RTO本来就有很大的随机性的元素，选择什么样的RTOmin，是一个取舍的问题。

```
As a result, wide-area “packet delays [are] not mathematically [or] operationally steady”, which confirms the Allman and Paxson observation that RTO estimation involves a fundamental tradeoff between rapid retransmission and spurious retransmissions.
```

.

Delayed ACK确实可能对器有一些影响，但是，这个情况可以被TCP的新功能和能产生影响的有限的条件缓和。说了好大的一堆，就是说没啥影响。

```
In practice, these two potential consequences are mitigated by newer TCP features and by the limited circumstances in which they occur, as we explore in the next two sections. We find that eliminating the RTOmin has little impact on bulk data transfer performance for wide- area flows, and that in the datacenter, delayed ACK causes only a small, though noticeable drop in throughput when the RTOmin is set below the delayed ACK threshold.
```

.

在广域网上测试的数据 ：

![rto-wide-erea](/assets/img/rto-wide-erea.png)



## 参考

1. Safe and Effective Fine-grained TCP Retransmissions for Datacenter Communication, SIGCOMM 2009.