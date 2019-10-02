---
layout: page
title: Cardinality Estimation Algorithms
tags: [Algorithm]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## A linear-time probabilistic counting algorithm for database applications

### 0x00 引言

   这里总结了一些Cardinality Estimation算法的思路。这类算法处理的问题是非精确的统计一堆数据中，里面不同对象的数量。在数据量不大的时候，使用Set(Based Hash or Tree)是一个不错的方式，但是在数据量非常大的时候，比如几百亿甚至是更多的时候，这些精确的统计算法显得不是很必要，极少有场景需要在数据在几百亿的量级上还要精确地统计到底有多少个。这个时候非精确的统计会是一个更加好的方法。这里最常用的算法就是Hyper Log Log，在Redis中也有提供，消耗的内存低到惊人，一般是10KB级别的消耗，误差也很低，简直就是一个几乎完美的算法。

### 0x01 基本思路

 这篇Paper不是Hyper Log Log，而是一种叫做Linear Counting的算法。它的基本思路还是利用bitmap，先将对象计算一个hash值，然后在这个bitmap上面，将对应的bit为设置为1，然后通过这个bitmap中bit为0的位数来统计元素的数量。设 N*为估计的对象的数量，u为bit位为0的数量，Vn = u / m，m为bitmap的长度。
$$
N^* = -m \ln V_n = -m\ln\frac{U_n}{m}
$$
 每次设置之后，一个bit不被设置的概率位1 - 1/m，n次的话就是 (1 - 1/m)^n，这里稍微改变一下这个式子，在m，n足够大的时候，这个式子可以变为,
$$
(1-\frac{1}{m})^n = (1-\frac{1}{m})^{-m\frac{n}{-m}} =  e^{-\frac{n}{m}}
$$
 这个是高数中求极限会常用到的一种方法。n个求和就是,
$$
E(U_n) = \sum_{i-1}^{m} P_A(i) = m\cdot e^{-\frac{n}{m}} \\
即 n = -m \ln \frac{E(U_n)}{m}
$$
 这里可以这么u位E(Un)为最大似然估计，即得到前面的式子。误差分析这里可以参看[1]。总体思路还是比较简单的，这里的空间复杂度还是O(n)。

## Loglog Counting of Large Cardinalities

### 0x10 基本思路

  Loglog Counting算法称之为Loglog Counting的原因是其内存消耗只有O(log(log(n))级别。这个只是数学意义上的，实际上应用O(log(log(n))级别的算法完全就可以认为是常数的内存消耗。这个算法的基本思路如下，1. 计算所有对象的hash值，记录下这个hash值从左到右(从右到左也是一样的)的第1个为1的位置。这里认为hash函数产生的hash值每个bit为0 or 1的概率是一样的，这里可以使用N* = 2^p，其中N\*为估计的数量，p为记录最右边的第一个为1的位置，从1开始。这里个实际上就是类似于抛硬币，连续p次向上的概率为1/2^p。

![](/assets/images/loglog-basic.png)

 如果只是用一个hash函数，这里的误差就会比较大，而且估计的值只会是2的次幂。这里的优化方式如上图的伪代码。可以将hash值拆分为几个部分，称之为bucket。这里M(j)使用的是这些值去平均的方式。这样得到估计
$$
E = a_m \cdot m \cdot 2^{\frac{1}{m}\sum_{j}M^{(j)}} \\
其中m为bucket的数量，而2^{\frac{1}{m}\sum_{j}M^{(j)}}是根据前面思路得出的数量\\
a_m 这里为误差修正，a_m = (Γ (−1/m)\frac{1-2^{\frac{1}{m}}}{log2})^{-m}
$$
 Γ为著名的伽马函数，修正参数的具体信息可以参看[2]。Loglog Counting的缺点是估计的数量比较小是误差比较大。

## HyperLogLog: the analysis of a near-optimal cardinality estimation algorithm

### 0x01 基本思路 

 HyperLogLog算法的出现是未来还将Log Log Counting算法的缺点。总体的算法的思路是一样的，也是分桶，求第一个bit的位置，然后利用这些信息计算一个值。这里的区别不是利用前面的各个bucket第一个bit位置的几何平均数，这里使用的是调和平均数。调和平均数的一个优点是对离群点的容忍程度更大，能够降低由于这个点来的估计误差。

![](/assets/images/hloglog-basic.png)

经过上面的调整之后，公式变为
$$
调和平均数计算H = (\frac{\sum_{i=1}^{n}x_i^{-1}}{n})^{-1} \\
E = a_m\cdot m^2\cdot (\sum_{j=1}^{m}2^{-M_{[j]}})^{-1}, 这里a_m 变为了\\
a_m = (m\int_0^\infty{(log_2{(\frac{2+u}{1+u}))^m du}})^{-1}
$$
 纠正误差的公式的具体推导可以参看[3]。但是这样的公式在具体实现的时候，会比较麻烦。实际使用的时候会采用一些近似的计算方式，Paper中也提出了一种一些解决方式。在数据量小的时候也利用LC的思路进行改进。

![](/assets/images/hloglog-alg.png)

## 参考

1. A linear-time probabilistic counting algorithm for database applications, TODS '90.
2. Loglog Counting of Large Cardinalities, 2003.
3. HyperLogLog: the analysis of a near-optimal cardinality estimation algorithm, AofA '07.

