---
layout: page
title: Several Congestion Control Algorithms
tags: [Network]
excerpt_separator: <!--more-->
typora-root-url: ../
---

>一些TCP拥塞控制算法的基本思路

## TCP Vegas: New Techniques for Congestion Detection and Avoidance

### 0x00 基本思路

  TCP Vegas也是一个很有名的TCP拥塞控制的算法。它和常见的NewReno，Cubic之类的基于丢包的算法不同，Vegas是一种基于延迟的算法。Vegas的一个主要的缺点就是在和这些基于丢包的算法一起运行的时候，它显得太过于“君子”，往往比这些算法提前推出窗口增长的阶段。

* 在Vegas中，有下面几个基本的概念，1. BaseRTT，Vegas算法中认为链路上面的基本的RTT时间，一般使用测量到的最小值(考虑到链路随时可能变化)，这里可能选选取一段时间内的最小值。然后根据当前的WindowSize，计算出Expected = WindowSize / BaseRTT，Expected即目标的吞吐。
* 另外，Vegas会测量Actual，即某一个时间的实际的吞吐的值。设Diff = Expected - Actual。如果有 Actual > Expected，则表明Vegas需要对BaseRTT进行更新。Diff这里只会是不小于0的数。另外，Vegas定义了两个值a、b，其中a < b。如果 Diff < a，则表明实际的吞吐和期望的差不多，则在下一个RTT中增加窗口大小。如何Diff > b，则表明实际的明显小于期望，则认为可以出现了拥塞，则尝试减小窗口的大小。在这两个值中间的时候保持当前的窗口大小。
* 另外Vegas也使用了在其它的一些算法中使用的快速重传的策略。

## BIC Congestion Control Algorithm

### 0x10 基本思路

  BIC算法是一种为长肥网络优化的TCP拥塞控制的算法。它处于这样的一个基本思路：在慢启动之后，之前的算法是进入一个加性增的过程(AIMD中的AI)。这个过程时间上就是在慢启动中最大的窗口和慢启动结束之后的窗口大小之间的一个搜索的过程。BIC的思路就是在这里使用二分查找的方法。由于长肥网络下面加性增的过程可能很长，BIC的这个思路能够让发送窗口的大小更加块地拟合到实际需要的大小。下面的伪代码是维基百科上面对BIC简单地描述，

```
//Smax:    the maximum increment
//Smin:    the minimum increment
//wmax:    the maximum window size  
//β:       multiplicative window decrease factor
//cwnd:    congestion window size  
//bic_inc: window increment per RTT (round trip time)

// 在这里使用了二分的策略，直接使用两个值之间中间的值
// One step of increasing cwnd:
 if (cwnd < wmax)          // binary search OR additive
   bic_inc = (wmax - cwnd) / 2;
 else                     // slow start OR additive
   bic_inc = cwnd - wmax;
  
// 这里的优化是为了增长过大 or 过小，设置了两个阈值。
 if (bic_inc > Smax)      // additive
   bic_inc = Smax;
 else if (bic_inc < Smin) // binary search OR slow start
   bic_inc = Smin;
 cwnd = cwnd + (bic_inc / cwnd);
 
 // One step of decreasing cwnd:
 if (cwnd < wmax) // fast convergence
   wmax = cwnd * (2-β) / 2;
 else 
   wmax = cwnd;
 cwnd = cwnd * (1-β);
```

## CUBIC: A New TCP-Friendly High-Speed TCP Variant 

### 0x20 基本思路

  CUBIC是从BIC算法的思路延伸而来的一个算法。基本思路是用一条三次方的曲线取代BIC中二分查找的过程。Cubic核心的一个函数是
$$
\\ W(t) = C(t - K)^3 + W_{max}, 其中 K = \sqrt[3]{\frac{W_{max}\beta}{C}}
$$
 其中C为一个参数，t是从上次窗口减小操作到现在的时间，K是要从曲线开始的窗口大小增长到Wmax所用的时间，Wmax是上次窗口减小操作的时候窗口的值。。另外Cubic的另外一个参数就是每次减小的因子β。从下面的图中可以看出来Cubic和Bic的窗口变化的曲线是很类似的。这个方程的来源可以从一下几个点推出来，1. 曲线中间的对称点事Wmax，2. 曲线有对称性，起点为上次需要进行窗口减小操作之后设置的值，3. 增长到Wmax的时间为K。通过这些条件就可以推出这个方程的来源。C、β分别影响了Cubic算法收敛的速度和TCP的公平性。

![cubic-functions](/assets/images/cubic-functions.png)

## A Compound TCP Approach for High-speed and Long Distance Networks

### 0x30 基本思路

  Compound TCP事Windos采用的拥塞控制的算法，它的基本思路就是将基于丢包的方法和基于延迟的方法结合起来。在计算机科学中结合不同策略来实现取长补短形成新的策略是很常见的方法。它的窗口的大小为win = min(cwnd + dwnd,awnd)，其中cwnd为传统的TCP方法中的cwnd，另外dwnd事基于延迟得到的窗口的大小，实际的窗口大小等于这两者之和。另外awnd表示的接受方提出的窗口大小。另外，

* CTCP在加性增，即拥塞避免阶段使用的计算方式是cwnd =cwnd+1/(cwnd+dwnd)，和传统的方法增加了dwnd。

* 在基于延迟的部分，基本上还是采用Vegas的策略，

  ```
  Expected = win / baseRTT
  Actual = win / RTT
  Diff = (Expected − Actual) * baseRTT
  An early conges- tion is detected if the number of packets in the queue is larger than a threshold γ . If diff <γ , the network path is determined as under-utilized; otherwise, the network path is considered as busy and delay-based component should gracefully reduce its window.
  ```

  另外也CTCP减少窗口的策略也是乘性减，参数也是deta。dwnd根据diff的值和丢包的情况变化公式如下。

  ![ctcp-dwnd](/assets/images/ctcp-dwnd.png)

## 参考

1. TCP Vegas: New Techniques for Congestion Detection and Avoidance, SIGCOMM 1994.
2. https://en.wikipedia.org/wiki/BIC_TCP，维基百科.
3. CUBIC: A New TCP-Friendly High-Speed TCP Variant. ACM SIGOPS Operating System Review, 2008. 
4. A Compound TCP Approach for High-speed and Long Distance Networks, INFOCOM, 2006.
5. FAST TCP: Motivation, Architecture, Algorithms, Performance. IEEE/ACM Trans. on Networking, 2006. 