---
layout: page
title: Datacenter TCP, Deadline Driven Delivery and Deadline-aware Datacenter TCP
tags: [Transport Protocol, Data Center,Network]
excerpt_separator: <!--more-->
typora-root-url: ../
---



## Datacenter TCP, Deadline Driven Delivery and Deadline-aware Datacenter TCP



### 0x00 引言

  这篇总结包含了3篇Paper的内容，一篇是SIGCOMM 2010上的DCTCP，一篇是SIGCOMM 2011上的Deadline Driven Delivery，还有一篇是SIGCOMM 2012上面的D2TCP。前者将的是如何利用**Explicit Congestion Notification** (**ECN**)解决数据中心网络中TCP的一些问题，第二个是如何加入deadline的优化，后者是前2者的优化。

  这里只是简单地介绍。



### 0x01 ECN

 ECN就是显示的拥塞通知。对于IPv4，它使用了DiffServ字段最右边的两个bits来标示(在一些早一点的书上，可以发现说这里是预留给以后的功能的，目前没有使用，当然现在是已经使用了)，在IPv6上Traffic Class字段的最后两个bits。

![dctcp-header](/assets/img/dctcp-header.png)

​    这里只给出了IPv4的header，图片来源于维基百科。

- `00` – 不支持ECN；
- `10` – 支持ECN；
- `01` – 支持ECN，和上面相同；
- `11` –遇到了阻塞；

ECN还有更多的细节，可参考相关资料。

.

### 0x02 问题

1. incast 问题，在Partition/Aggregate模式中比较常见，服务器同时回复请求端导致某个地方的包突然大量增加，从而导致丢包；
2. 排队问题，长时间的流和短时间的流同时使用一个交换机端口时，导致排队，也导致短时间的数据被drop，及时没有被drop页导致了延时的增加；
3. buffer的问题，不同的流使用不同的交换机短空，长时间的流占用了共享的buffer。

![dctcp-problems](/assets/img/dctcp-problems.png)



### 0x03 DCTCP

  DCTCP利用了ECN。在交换机上，当一个端口的包超过一定的阈值之后，给包加上ECN标志。包的接受这在接受到这些包之后，将这些信息回复给发送者。发送者根据收到的包里面的ECN的情况来调整发送行为。这样就可以在发生拥塞之前就调整行为。总结一下就是3个部分:

1. Simple Marking at the Switch；
2. ECN-Echo at the Receiver；
3. ControllerattheSender；

发送方记录收到的回复包里面比例，使用这样一个函数更新：

```
α ← (1 − g) × α + g × F
```

F为最新窗口里面被标记的笔记，g时一个0到1之间的比例值，这个和RTT的时间估计类似。

然后使用:

```
cwnd ← cwnd × (1 − α/2)
```

来更新cwnd。

.

效果的部分数据:

![dctcp-results](/assets/img/dctcp-results.png)



### 0x04 Deadline Driven Delivery 

  D3的出现时为了解决DCTCP中的deadline的问题，使用的时带宽分配的方式。每一个RTT内，发送方都计算需要在deadline之前发送完数据的带宽，然后把这个信息放进包里面。交换机在收到了这样的信息之后，使用贪婪的方式分配带宽：

 对于有deadline的流，就在平均共享的带宽上加上发送方需要的带宽的值，没有，则就选择平均分配的带宽:

```
• For a deadline flow with desired rate r, a = (r+fs), where fs is the fair share of the spare capacity after satisfying deadline flow requests.
• For a non-deadline flow, a = fs.
```

这里的具体操作还有更多的细节:

```
The rate allocation description above assumes the router has the rate requests for all flows at the same point in time. In reality, the router needs to make allocation decisions in an online, dynamic setting, i.e., rate requests are spread over time, and flows start and finish. To achieve this, the rate allocation operates in a slotted fashion (from the perspective of the endhosts). 
```

结果的部分数据：

![dctcp-d3-results](/assets/img/dctcp-d3-results.png)





### 0x05 D2TCP

  DCTCP也是为了解决DCTCP中不可值的deadline时间改进的，同时解决D3种存在的问题。它处理考虑到拥塞的情况性外，还考虑了包的deadline信息，到达了以下的效果：

```
• reduces the fraction of missed deadlines compared to DCTCP and D3 by 75% and 50%, respectively;
• achieves nearly as high bandwidth as TCP for background flows without degrading OLDI performance;
• meets3deadlines that are 35-55% tighter than those achieved by D for a reasonable 5% of missed deadlines, giving OLDIs more time for actual computation; and
• coexists with TCP flows without degrading their performance.
```

D2TCP的Congestion Avoidance算法，首先同样时DCTCP中的一个公式:

```
α = (1 − g) × α + g × f
```

 然后定义一个参数d，代表了deadline的紧迫程度，这里的d越大代表越紧迫，然后计算一个参数p(就是penalty的意思)：

```
p = a ^ d
```

```
This function was originally proposed for color correction in graphics, and was dubbed gamma-correction because the original paper uses γ as the exponent. Note that being a fraction, 𝛼 ≤ 1 and therefore, 𝑝 ≤ 1. 
```

然后就计算新的窗口大小:

```
w = w * (1 - p/2) if p > 0,
  = w + 1, if p = 0
```

 这里可以看出来，如果没有被标记的包，a = 0，这样p就是0，行为和正常TCP的行为一样，如果a = 1，那么计算处理啊的w就是正常情况下的一半。具体的d如何处理可参加论文。

 Famma-correction函数的示意图:



![dctcp-correction](/assets/img/dctcp-correction.png)

  具体分析这里省略了。结果的部分数据:

![d2tcp-results](/assets/img/d2tcp-results.png)





.

## 参考

1. Data Center TCP (DCTCP), SIGCOMM 2010.
2. Deadline-Aware Datacenter TCP (D2TCP), SIGCOMM 2012.
3. Better Never than Late: Meeting Deadlines in Datacenter Networks, SIGCOMM 2011.
4. https://en.wikipedia.org/wiki/Explicit_Congestion_Notification, Explicit Congestion Notification (ECN).