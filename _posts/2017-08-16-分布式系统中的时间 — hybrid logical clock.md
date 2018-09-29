---
layout: page
title: Hybrid Logical Clock
tags: [Distributed]
excerpt_separator: <!--more-->
typora-root-url: ../
---



# 分布式系统中的时间 — hybrid logical clock
  发布系统中的时间有很重要的作用，比如Spanner的TrueTime[2]。但是TrueTime实现的难度太大，就算开源社区实现了，也不太好被一般的用户使用。所以CockRoachDB使用了Hybrid Logical Clock(HLC)，HLC是一种Logical Clock的实现，HPC将Logical Clock和物理时钟联系起来，与物理时间之间的误差在一个固定的值之内，这个值由NTP决定。

## 介绍
  分布式系统中有几个关于时间的概念：
    1. Logical clock (LC)：逻辑时间是有Lamport提出的，LC独立于物理上的时间存在。
    2. Physical Time (PT)：
    3. TrueTime (TT)：TT出现的时间比较近，在Google的Spanner中被使用。目前看来，这个最有B格。
    4. HybridTime (HT)：


## 背景
  对于由一系列可能随时间变化的节点组成的分布式系统，每个节点可以执行3中操作：1.发送动作，2. 接收动作和3.本地动作。时间戳算法为每个时间分配时间戳。如果使用LC算法来分配时间戳，则给时间e分配的时间记为 lc.e。


## HLC
 HLC的设计目标是提供像LC一样的单向因果检测，同时保持时钟的值总是接近物理的时间（这里是NTP的时间）。一个HLC的表述如下：
  给定一个分布式系统，为每一个事件分配一个时间戳，有：
    1. e hb f ⇒ l.e < l.f，e事件happen before f，则有l.e < l.f;   
    2. Space requirement for l.e is O(1) integers, l.e消耗一个整数的空间；   
    3. l.e is represented with bounded space, l.在一个有界的空间内比表示；  
    4. l.e is close to pt.e, i.e., |l.e − pt.e| is bounded. l.e接近pt.e ,|l.e - pt.e|是有界的。
     （pt -> Phiysical Time)。


### 基本的HLC算法
  为了实现以上的4点，这里有一个最简单的实现：
```
Initially lc.j := 0 

Send or local event 
  l.j := max(l.j + 1, pt.j) # 发送是l.j设置为l.j+1 与 物理时间pt.j 的较大值
  Timestamp with l.j 
  
Receive event of message m   l.j := max(l.j + 1, l.m + 1, pt.j)  # 接受到时时间戳设置为 l.j+1, 接受到的时间戳+1与物理时间pt.j中的较大值
  Timestamp with l.j 
```

以上的基本算法时满足1，2的，乐死基本的lamport clokc加入了pt的元素。但是这个算法并不能满足4条中的3、4条。|l.e − pt.e|可能会无限的增长。原因在于每一个操作都有可能增大l.e与pt.e的差距。

### HLC Algorithm 
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
  Timestamp with l.j, c.j ```
HLC的变化注意有3点：（1）去除了+1的操作，（2）增加了c，（3）c有reset的逻辑。
  max()的计算逻辑满足了l.j >= pt.j 的，可以理解为l.j保存了节点j已知的系统中的最大的pt值。
  l.j由于没有了+1的操作，不会出现l.j比pt越来越大的情况。而对于有+1操作的c，c的出现又是为什么呢？
  c出现的原因时由于没有了+1的操作，有可能导致了l.e = l.f，为了解决这个问题引入了c。这里就使用使用⟨l.e, c.e⟩ < ⟨l.f, c.f ⟩ 来表示先后的关系。同时，为了避免c的无限增长，当我们知道目前有l.e < l.f 时，可以将c reset为0。在算法中，就是本节点的pt时max是，c会被reset为0。

## 实现
CockroachDB中就是实现了HLC算法[3]。实现HLC的难度不大，代码也比较少，这里指给出关键的两个函数，具体的信息可以参考代码。
Send，这里和上面的伪代码是相同的逻辑：

​```go
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

Recv的逻辑也是相同的：
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

## 参考
1. Logical Physical Clocks and Consistent Snapshots in Globally Distributed Databases：http://www.cse.buffalo.edu/tech-reports/2014-04.pdf
2. Spanner: http://research.google.com/archive/spanner-osdi2012.pdf
3. CockroachDB HCL Algorithm: https://github.com/cockroachdb/cockroach/blob/master/pkg/util/hlc/hlc.go


   ​			
   ​		
   ​	


​			
​		
​	
