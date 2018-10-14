---
layout: page
title: Distributed 0x01
tags: [Distributed]
excerpt_separator: <!--more-->
typora-root-url: ../
---



## Distributed 0x01

 这里先来讨论分布式系统中物理时间。首先有一下的几点：
    1. 相对论中，由于存在多个参考系，因此时间测量可能是不准确的。当然，地球上的分布式系统中，相对型的影响是可以忽略的。
    2. 即使这样，受目前技术能力的限制，我们还是不能准确记录不同点上的事件发生的时间，以此表明事件发生的顺序。

我们将两个时钟读数之间的瞬间不同被称为始终漂移（clock skew）。时钟漂移以单个时钟读数和完美的参考时钟之间的漂移来度量。此外，参考时钟度量的每个单位时间内，和完美的参考时钟之间的漂移量称为漂移率。目前，原子钟时漂移率最小的时钟（Spanner中就使用了原子钟）。	
原子时钟的输出被用作实际时间的标准，称为国际原子时间，而秒、年等我们使用的时间单位来源于天文时间，与原子时间并不一致，通用协调时间（UTC）是国际计时标准，它基于原子时间的，但是偶尔需要增加闰秒或极偶尔的情况下要删除闰秒（这个闰秒导致了很多的软件故障）[1]。



### 同步物理时钟

  为了知道分布式系统P的进程中事件发生的具体时间，有必要用权威的外部时间源同步进程的时钟Ci -- 外部同步（external synchronization）。
	如果时钟Ci与其他时钟同步到一个已知的精度，那么我们就能通过本地时钟度量在不同计算机上发生的两个事件的间隔  -- 内部同步（internal synchronization）。
	这里要定义一下时钟的正确性。正如我们所知，完全准确的时钟时几乎不可能的，所以：	 时钟正确性（correctness）通常定义为，如果一个硬件时钟H的漂移率在一个已知的范围ρ>0内，那么该时钟是正确的。
	  1. 这表明度量实际时间t和t’的时间间隔的误差是有界的
```
                     (1- ρ)(t’-t) ≤ H(t’)-H(t) ≤ (1+ ρ)(t’-t)
```
	  2. 该条件禁止了硬件时钟值的跳跃，有时，软件也时钟也要求遵循该条件。但是用一个较弱的单调性条件就足够了。
	  3. 单调性是指一个时钟C前进的条件
```
                 t’>t ==> C(t’) > C(t)
```

  现在来看看时钟同步中的一些情况:
   一个进程在消息m中将本地时钟的时间发送给另一个进程:
      最简单的做法就是接收进程可以将它的时钟设成t+T(trans)，但是T(trans)是不确定的。
   在*同步的*系统中：
  * 设消息传输时间的不确定性为u，那么有u=(max-min)；
  * 如果接收放将时钟设置为t+min，那么时钟偏移至多为u；
  * 如果接收放将时钟设置为t+max，那么时钟偏移至多为u；
  * 如果设置为t+(max+min)/2，那么是时钟偏移至多为u/2；
  * 同步系统要同步N个时钟，可获得的时钟偏移最优范围是u(1-1/N)。

然而在异步的系统中，消息传输的事件延迟是没有上界的，只有：T(trans) = min + x, x >= 0;



### Cristian方法

  Cristian Algorithm使用一个时间服务器，它连接到一个接收UTC信号的设备上，用于实现外部同步在接收到请求后，服务器S根据它的时钟提供时间。
  Cristian Algorithm:
    1. 一个进程P发送一个请求信号给时间服务器S；
    2. S收到P的请求包之后，在包上面加上当前S的时间，然后回复P；
    3. P收到回复后，将P的当前时间设为T + RTT/2。
       此外 Cristian Algorithm通过给S发送几个请求，Tround最小值给出最精确的估计。
         Cristian Algorithm基于RTT的，要求RTT的准确性。此外，Cristian Algorithm存在单点的问题。



### Berkeley Algorithm

  Berkeley Algorithm与Cristian Algorithm不同。Cristian Algorithm是用在一个客户端向一个服务器请求时间。而Berkeley Algorithm是几个客户端之间同步时钟。
  算法的主要步骤:
	1. 选择一台协调者计算机作为master；
	2. Master定期轮询其他要同步时钟(slave)的计算机；
	3. Slave将它们的时钟值返回给主机；
	4. Master通过观察往返时间来估计它们的本地时钟时间，并计算所获得值的平均值，所以协议的准确性依赖于主从机之间的名义上最大往返时间；
	5. 主机发送每个从属机的时钟所需的调整量。
	
  这里要注意的是Slave受到了调整量后，如果需要把时钟往回调，则一般是通过让时钟变慢一点，而不是真的回调，因为在很多的程序中，都以时间只会往前走而不会后退为基本假设，后退时钟可能导致一些程序的Bug。



### NTP

  上面说的两种Cristian方法和Berkeley算法主要应用于企业内部网，而网络时间协议（Network Time Protocol，NTP）定义了时间服务的体系结构和在互联网上发布时间信息的协议。
  NTP服务由互联网上的服务器网提供，主服务器（primary server）直接连接到像无线电时钟这样的接收UTC源，二级服务器（secondary server）与主服务器同步。
  NTP有3中模式，所以模式都以UDP传输：
	1. 组播模式，用于高速LAN，一个或多个服务器定期将时间组播到由LAN连接的其他结点，并设置它们的时间。需要基于延迟很小这一假设。
	2. 过程调用模式（procedure-call mode），类似与Cristian算法，服务器从其他计算机接收请求，并用时间戳应答
	3. 对称模式（symmetric mode），用于在LAN中提供时间信息的服务器和同步子网的较高层，一对服务器交换有时序信息的消息。时序数据作为服务器之间的关联的一部分被保留，时序数据可用于提高时间同步的准确性

  过程调用模式和对称模式中，进程交换消息对，每个消息有最近消息的时间戳: 发送和接收前一个NTP消息的本地时间，发送当前消息的本地时间。NTP消息的接收者记录它接收消息的本地时间。

 对于两个服务器之间发送的每对消息，由NTP计算偏移 o(i) 和延迟 d(i)
	 * 偏移o(i)是对两个时钟之间实际偏移的一个估计
	 * 延迟d(i)是两个消息整个的传输时间
```
Server A
+-------------------T(i-2)----------------T(i)---+


+-----------T(i-3)-------------T(i-1)------------+
Server B
```
 如果B上的时钟相对于A的真正偏移是o，而m和m’实际的传输时间分别为t和t’，那么我们可以得到：
```
          T(i-2) = T(i-3) + t + o
          T(i) = T(i-1) + t’- o
```
可以推出：
```
	     d(i) = t + t’ = T(i-2) –T(i-3) + T(i) – T(i-1)
		  o = o(i) + (t’-t)/2，其中 o(i) = (T(i-2) – T(i-3) + T(i-1) – T(i)) / 2
```
 利用t和t’≥0的事实，有 o(i) - d(i)/2 ≤ o ≤ o(i) + d(i)/2。o(i)是偏移的估计，d(i)是该估计的精确性度量.

.

### Hybrid Logical Clock

  CockRoachDB使用了Hybrid Logical Clock(HLC)，HLC是一种Logical Clock的实现，HPC将Logical Clock和物理时钟联系起来，与物理时间之间的误差在一个固定的值之内，这个值由NTP决定。

  对于由一系列可能随时间变化的节点组成的分布式系统，每个节点可以执行3中操作：1.发送动作，2. 接收动作和3.本地动作。时间戳算法为每个时间分配时间戳。如果使用LC算法来分配时间戳，则给时间e分配的时间记为 lc.e。



### HLC

 HLC的设计目标是提供像LC一样的单向因果检测，同时保持时钟的值总是接近物理的时间（这里是NTP的时间）。一个HLC的表述如下：
  给定一个分布式系统，为每一个事件分配一个时间戳，有：

1. e hb f ⇒ l.e < l.f，e事件happen before f，则有l.e < l.f;   
   1. Space requirement for l.e is O(1) integers, l.e消耗一个整数的空间；   
   2. l.e is represented with bounded space, l.在一个有界的空间内比表示；  
   3. l.e is close to pt.e, i.e., |l.e − pt.e| is bounded. l.e接近pt.e ,|l.e - pt.e|是有界的。

```
 （pt -> Phiysical Time)。
```

.

#### 基本的HLC算法

  为了实现以上的4点，这里有一个最简单的实现：

```
Initially lc.j := 0 

Send or local event 
  l.j := max(l.j + 1, pt.j) # 发送是l.j设置为l.j+1 与 物理时间pt.j 的较大值
  Timestamp with l.j 
  
Receive event of message m   l.j := max(l.j + 1, l.m + 1, pt.j)  # 接受到时时间戳设置为 l.j+1, 接受到的时间戳+1与物理时间pt.j中的较大值
  Timestamp with l.j 
```

以上的基本算法时满足1，2的，类似基本的lamport clokc加入了pt的元素。但是这个算法并不能满足4条中的3、4条。|l.e − pt.e|可能会无限的增长。原因在于每一个操作都有可能增大l.e与pt.e的差距。

.

#### HLC Algorithm

HLC算法的描述如下：

```
Initially l.j := 0; c.j := 0 

Send or local event 
  l′.j := l.j;   l.j := max(l′.j, pt.j); # 相比于最基础的去处了+1   If (l.j = l′.j) then 
    c.j := c.j + 1 
  Else 
    c.j := 0; 
  Timestamp with l.j, c.j 
  
Receive event of message m   l′.j := l.j;   l.j := max(l′.j, l.m, pt.j); # 相比于最基础的去处了+1   If(l.j=l′.j=l.m) then
    c.j := max(c.j, c.m) + 1 
  Else if (l.j =l′.j) then 
    c.j := c.j + 1 
  Else if (l.j = l.m) then 
    c.j := c.m + 1 
  Else c.j := 0 
  Timestamp with l.j, c.j
  
```
  HLC的变化注意有3点：（1）去除了+1的操作，（2）增加了c，（3）c有reset的逻辑。max()的计算逻辑满足了l.j >= pt.j 的，可以理解为l.j保存了节点j已知的系统中的最大的pt值。l.j由于没有了+1的操作，不会出现l.j比pt越来越大的情况。而对于有+1操作的c，c的出现又是为什么呢？c出现的原因时由于没有了+1的操作，有可能导致了l.e = l.f，为了解决这个问题引入了c。这里就使用使用⟨l.e, c.e⟩ < ⟨l.f, c.f ⟩ 来表示先后的关系。同时，为了避免c的无限增长，当我们知道目前有l.e < l.f 时，可以将c reset为0。在算法中，就是本节点的pt时max是，c会被reset为0。

##### 实现
CockroachDB中就是实现了HLC算法[3]。实现HLC的难度不大，代码也比较少，这里指给出关键的两个函数，具体的信息可以参考代码。
Send，这里和上面的伪代码是相同的逻辑：

```go
// c.getPhysicalClockLocked()还会检查时间jump（跳变）

func (c *Clock) Now() Timestamp {
	c.mu.Lock()
	defer c.mu.Unlock()

	if physicalClock := c.getPhysicalClockLocked(); c.mu.timestamp.WallTime >= physicalClock {
		// 当WallTime大于物理时间是，Logical++
		c.mu.timestamp.Logical++
	} else {
		// 否则使用物理时间病Logical置0
		c.mu.timestamp.WallTime = physicalClock
		c.mu.timestamp.Logical = 0
	}
	return c.mu.timestamp
}
```

Recv的逻辑也是差不多的：
```go
func (c *Clock) Update(rt Timestamp) Timestamp {
	c.mu.Lock()
	defer c.mu.Unlock()
	physicalClock := c.getPhysicalClockLocked()

	if physicalClock > c.mu.timestamp.WallTime && physicalClock > rt.WallTime {
		// 当物理时间最大的，取物理时间，Logical置0,
		// 对应了上面伪代码的最后一个else
		c.mu.timestamp.WallTime = physicalClock
		c.mu.timestamp.Logical = 0
		return c.mu.timestamp
	}
	
	// 两个WallTime不想等则取大的一个，同时Logical++
	if rt.WallTime > c.mu.timestamp.WallTime {
		offset := time.Duration(rt.WallTime-physicalClock) * time.Nanosecond
		if c.maxOffset > 0 && offset > c.maxOffset {
			log.Warningf(context.TODO(), "remote wall time is too far ahead (%s) to be trustworthy - updating anyway", offset)
		}
		c.mu.timestamp.WallTime = rt.WallTime
		c.mu.timestamp.Logical = rt.Logical + 1
	} else if c.mu.timestamp.WallTime > rt.WallTime {
		c.mu.timestamp.Logical++
	} else {
		// 两个WallTime想等则Logical取较大的一个，然后Logical++
		// 对应了上面伪代码的第一个If
		if rt.Logical > c.mu.timestamp.Logical {
			c.mu.timestamp.Logical = rt.Logical
		}
		c.mu.timestamp.Logical++
	}
	return c.mu.timestamp
}
```

.

## 参考

1. COMP130123 Distributed Systems [Fall 2015]: http://jkx.fudan.edu.cn/~qzhang/COMP130123.html.
2. Logical Physical Clocks and Consistent Snapshots in Globally Distributed Databases：http://www.cse.buffalo.edu/tech-reports/2014-04.pdf
3. Spanner: http://research.google.com/archive/spanner-osdi2012.pdf
4. CockroachDB HCL Algorithm: https://github.com/cockroachdb/cockroach/blob/master/pkg/util/hlc/hlc.go

