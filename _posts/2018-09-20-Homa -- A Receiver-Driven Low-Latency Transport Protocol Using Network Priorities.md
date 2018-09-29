---
layout: page
title: A Receiver-Driven Low-Latency Transport Protocol
tags: [Transport Protocol, Data Center, Network]
excerpt_separator: <!--more-->
typora-root-url: ../
---



## Homa -- A Receiver-Driven Low-Latency Transport Protocol Using Network Priorities 



## 0x00 引言

 最近几年为数据中心设计的新的传输协议不少，这篇时SIGCOMM上最新的一篇(截止写这篇论文时)，总而言之，这篇论文做到了:

```
In simulations, Homa’s latency is roughly equal to pFabric and significantly better than pHost, PIAS, and NDP for almost all message sizes and workloads. Homa can also sustain higher network loads than pFabric, pHost, or PIAS.

-----

Our implementation of Homa achieves 99th percentile round trip latencies less than 15 μs for small messages at 80% network load with 10 Gbps link speeds, and it does this even in the presence of competing large messages. Across a wide range of message sizes and work- loads, Homa achieves 99th percentile latencies at 80% network load that are within a factor of 2–3.5x of the minimum possible latency on an unloaded network. 
```

Homa倾向于为short messages设计。



### 0x01 KEY IDEAS 

Homa的设计有4个key design principles ：

```
(i) transmitting short messages blindly;

(ii) using in-network priorities;

(iii) allocating priorities dynamically at receivers in conjunction with receiver-driven rate control;

(iv) controlled overcommitment of receiver downlinks.
```

Homa这么设计出于一下的考虑：

```
1. There is no time to schedule every packet. 
```

 特别在意延时的情况下，schedule带来的延时都不可以接受。所以有了principle 1。



```
2. Buffering is a necessary evil.
```

没有一个protocol在没有导致buffering的同时实现low latency，讽刺的是，buffering又会带来latency。buffer，latency，throughput，欢喜冤家，emmmmmmm这里是不是可以写一本书了。



```
3. In-network priorities are a must. 
```

由于上一条，为了减少延时，不同的packet区分处理能获得一些效果。这个方法在很多类似的protocol中都有使用。这样就有了principle 2.



```
4. Making best use of limited priorities requires receiver control.
```

 简而言之就是recevier控制更加好。



```
5. Receivers must allocate priorities dynamically.
```

 Homa 使用Receivers动态分配优先级的方式解决了之前类似协议的一些问题(pHost )，比如large的messag使用高优先级带来的问题，只使用一种优先级可能导致的delay。这样就有了principl 3。



```
6. Receivers must overcommit their downlink in a controlled manner.
```

为了解决一些情况下链路利用率低的问题，比如一个sender项多个recevier发送数据(注意Homa使用的是receiver控制的传输方式)。为了解决这个问题，一个receiver可以过量使用downlink，比如同时给几个sender发送可以向receiver发送数据的grants。这样可能造成packet queuing ，但是对于提高利用率来说是必要的。



```
7. Senders need SRPT(shortest remaining processing time first) also. 
```

 排队也可能在sender端出现，Sender知道SRPT能跟好的解决这些问题。



### 0x02 基本设计

先来一张论文中的图：

![homa-arch](/assets/img/homa-arch.png)

Homa有以下特点：

```
Homa contains several unusual features: 
it is receiver-driven; 
it is message-oriented, rather than stream-oriented; 
it is connectionless; 
it uses no explicit acknowledgments; 
and it implements at-least-once semantics, rather than the more traditional at-most-once semantics.
```





#### RPCs, not connections 

 Homa是无连接的，一个来讲client的request message 对应一个来自server的 response message。由一个全局唯一的RPCid表示(id由客户端生成)。有以下的packet类型：

![homa-packet-types](/assets/img/homa-packet-types.png)



#### Basic sender behavior 



#### Flow control 



#### Packet priorities 







## 参考

1. Homa: A Receiver-Driven Low-Latency Transport Protocol Using Network Priorities, SIGCOMM 2018
2. 

