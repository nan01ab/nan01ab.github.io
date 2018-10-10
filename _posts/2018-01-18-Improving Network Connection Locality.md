---
layout: page
title: Improving Network Connection Locality on Multicore Systems
tags: [Operating System, Network]
excerpt_separator: <!--more-->
typora-root-url: ../
---



## Improving Network Connection Locality on Multicore Systems 



### 引言

  这个是最近三篇关于Linux 内核网络栈优化文章中的第1篇，后面的两篇会之后加上[2,3]。这篇文章解决Linux中Connection Locality的问题的。提出了一个叫做MegaPipe办法。

  问题:  这以幅图就基本说明了目前存在的问题。多个核心共享一些结构导致了冲突的问题。

![conn-local-listen](/assets/img/conn-local-listen.png)



### 基本思路

  基本思路就是将这些共享的结构弄成非共享的。Affinity-Accept 将这些结构变成了每一个核心一份。

  第一个要解决的问题就是包路由的问题，这里需要将来自一个连接的包有交给同一个核心。这里通过网卡的功能解决。

```
The NIC hardware typically hashes each packet’s flow identifier five-tuple (the protocol number, source and destination IP addresses, and source and destination port numbers) and uses the resulting hash value to look up the RX DMA ring where the packet will be placed. Since all packets from a single connection will have the same flow hash values, the NIC will deliver all packets from a single connection into a single DMA ring.
```

Accept Affinity每个core一个accept queue，应用线程优先在本地的core上的accept queue来处理连接。一个应用线程调用accept的时候，它返回本地的accpet queue上面的已经处理好的连接，如果本地没有，那就将这个线程睡眠。

```
When new connections arrive, the network stack wakes up any threads waiting on the local core’s accept queue. This allows all connection processing to occur locally.
```



### 负载均衡

 上面的分区的方式可能导致的一个问题就是负载不均衡的问题。这里的负载不均衡分为两类：

  这里就不具体讨论细节了。

* short-term imbalance，可能是一些突发的原因导致的，这里使用的解决方式就是connection stealing，类似于work steaing的功能。

  ```
  Affinity-Accept’s connection stealing mechanism consists of two parts: the first is the mechanism for stealing a connection from another core, and the second is the logic for determining when stealing should be done, and determining the core from which the connection should be stolen.
  ```

  

* longer-term load imbalance，可能是不均衡的分配导致的，这里的解决方式是 flow group migration 方式解决。

  ```
  each non-busy core finds the victim core from which it has stolen the largest number of connections, and migrates one flow group from that core to itself (by reprogramming the NIC’s FDir table). In our configuration (4,096 flow groups and 48 cores), stealing one flow group every 100ms was sufficient. Busy cores do not migrate additional flow groups to themselves.
  ```

.

>

### 评估

![conn-local-evolution](/assets/img/conn-local-evolution.png)





## 参考

1. Improving Network Connection Locality on Multicore Systems, EuroSys’12.
2. MegaPipe: A New Programming Interface for Scalable Network I/O, OSDI 2012.
3. Scalable Kernel TCP Design and Implementation for Short-Lived Connections, ASPLOS ’16.

