---
layout: page
title: Statistics in Orcacle Database
tags: [Operating System, Virtualization]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## Efficient and Scalable Statistics Gathering for Large Databases in Oracle 11g

### 0x00 基本内容

 这篇Paper主要的内容是Oracle数据一些统计方式的算法，主要讨论的是在分区表的情况下如何统计一些表的信息。Paper中描述的统计信息主要是一列中不同值的数量，类似于HyperLogLog做的事情。数据库统计主要用于优化器，一般有row sampling和block sampling的策略，前者更加接近于随机的取样，但是可能造成读取更多的数据，而后者随机性差一些，可能需要读取的数据更少。这里使用One-Pass Distinct Sampling的方式来统计一列不同值的数量。基本的算法比较容易理解，和HyperLogLog的思路有一些相同的地方。这里使用的算法也是在之前的算法上面优化而来的。基本的算法表示如下：

* 算法维护一个暂存的集合S，一个depth值d。对于每一个数据，计算其hash值h。如果hash值的前面d个都为0(实现的时候一般就是一个64bit hash的前面d bit都为0)，且h不在s中。如果S的大小达到了一个阈值N，则将d自增1。移除此时d位hash为1的对象，否则将h加入到S中。最终的结果为2^d * S.

* Paper中对于这个算法的精确度给出了证明，表示为
  $$
  \\ error ≈ \frac{\alpha}{s}\sqrt{s\cdot 2^d(1-2^{-d})}≈\frac{\alpha\cdot 2^{d/2}}{\sqrt{s}}.\\
  $$
  如果将s设置为2^d * N/2，可得
  $$
  \\ error ≈ \frac{\alpha\cdot 2^{d/2}}{\sqrt{2^d\cdot N/2}} = \frac{\alpha\sqrt{2}}{\sqrt{N}}.
  $$
  其中s为实际不同值的数量，N暂存几个的大小，a为统计标准差，variance表示为$ s\cdot 2^d(1-2^{-d}) $.

  ```
  For example, for N = 8000 we have at most 3% error with about 95% confidence (3% error means that the estimate sˆ is between 0.97 and 1.03 times the true value s), or at most 5% error with over 99% confidence. This corresponds for values of α = 2 and α = 3 respectively.
  ```

![](/assets/png/stats-alg.png)

  这个算法出自[2]。。在分区表的情况下，这里希望可以统计每个分区的数据，然后通过每个分区就能得到总的数据。所以这里需要一个merge统计信息的方法。这里将这个方法称之为 Synopsis Aggregation algorithm：d-max设置为m个分区中最大的d，在对各个分区的S中的hash值进行上门的NDV算法即可，

```
If we use the same memory size for all partitions and the Synopsis Aggregation algorithm then we get the same result as if we ran Approximate NDVs on the entire table (as if it was a single partition), with the same memory size. If we use more memory for the final stage (i.e., Synopsis Aggregation) we can potentially increase the accuracy of the algorithm.
```

#### 增量维护

  在得到了初始的统计信息之后，后面的统计使用增量维护的方式，只需要更新被认为需要更新的分区的统计数据。操作的第一步是发现有被DDL语句改动的分区，这里Paper中描述的是对分区的adding, dropping, coalescing, merging等操作。比如split一个分区被视为drop一个然后添加两个。第二步是发现被DML语句改动足够数量的分区。对于上面操作方向的分区，会重新收集统计信息。这个算法看起来未必比得上HLL。

## Adaptive Statistics in Oracle 12c

### 0x10 基本内容

 这篇Paper描述的是Oracle数据库中动态的一些特性。Oracle使用statistics queries来获取一些统计信息，Paper中举例了一些操作。这里使用adaptive sampling来实现更好的获取一些统计信息。Adaptive sampling基本的作用是对于一个table T和一个对于T的操作，通过采样的方式估计操作会产生的结果集合，

```
  Formally, the adaptive sampling addresses the following problem: given a table T and a set of operators applied to T, provide an estimate of the cardinality of the resulting dataset, based on a sample. The operators applied to T include table filters, joins, group by etc.
```

 其主要的内容分为以下的几个步骤：

* Sample，随机地选择n个block，然后在这些block上面应用操作。
* Cardinality Estimate，通过采样中选择block来获取到操作的resulting cardinality来估计整个数据库的这个操作的cardinality。
* Quality Test，估计关于一个Cardinality Estimate可信度区间，并对其进行一个quality test。
* Next Sample Size Estimate，如果上面的quality test测试通过，则操作可以停止了。如果不通过计算出一个下一次采样的block数量n-next，重复上面的操作。

### 0x11 基本设计

 Paper较大的篇幅来描述Adpative Sampling的方式以及其数学证明。对于这里的Adpative Sampling，要实现这样的几个目标：1. Cardinality Estimate and Confidence Interval，给出一个M的无偏估计M^，并给出其95%的上下界，M-l，M-u；2. Quality Test，对于一个λ，检测M-u <= (1+ λ) \* M^。对于λ=1的情况下，即有95%可信度来说M-u <= 2 \* M^；3. Next Sample Size Estimation，即估计n-next。总结来说就是在一个可信度的情况下估计一个比例的数据处于哪一个范围以内。

* 使用u表示评估每个block符合query条件的row的数量，即 per block query cardinality。这样在B个Block的table中，M= B \* u。假设经过K轮的采样，获取了N个Block，每个Block符合查询条件的row极为x-i，则u可以被估计为：
  $$
  \widehat{u} = \frac{\sum_{i=1}^N x_i}{N}\\
  $$
  根据中心极限定理，有u估计值符合的分布为一个正态分布，
  $$
  \widehat{u} = N(u, \frac{\sigma^2}{N}).\\
  \text{根据正态分布特性，u的100(1-α)% upper confidence bound 是}\\
  u_{UB}=\widehat{u}+z_{\alpha}*\frac{\sigma}{N}.
  $$
  其中z-α为z-score。这里使用α=0.025，这样z-α = 1.96。

* 要得到confidence interval，需要得到方差。根据前面的采样，对于完成K轮采样的情况下，K >= 2。第i轮采样符合查询的rows数量纪录为s-i，则
  $$
  \widehat{u} = \frac{\sum_{i=1}^K s_i}{\sum_{i=1}^K n_i} \\
  \widehat{M}的一个无偏估计可以表示为\widehat{M} = \widehat{u}*B.\\
  前面的x_i有可以表示为x_i = \frac{s_i}{n}，x_i同样是一个正态分布X_i = N(u, \sigma_i^2), \sigma_i=\frac{\sigma}{\sqrt{n_i}}。\\
  这样\frac{X_i-u}{\sigma_i}为一个标准正态分布。
  $$
  这样经过前面K轮的采样，可以得出一个自由度为K的卡方分布，
  $$
  \sum_{i=1}^K(\frac{X_i-u}{\sigma_i})^2\sim\chi_K \\
  \Rightarrow \sum_{i=1}^K n_i(\frac{X_i-u}{\sigma})^2\sim\chi_K \\
  \Rightarrow \frac{1}{\sigma^2}\sum_{i=1}^K n_i(X_i-u)^2\sim\chi_K \\
  这样在\beta为一定值, \sigma^2的上界为\widehat{\sigma}^2_{UB}, 有\\
  \Rightarrow \frac{1}{\widehat{\sigma}_{UB}^2}\sum_{i=1}^K n_i(X_i-u)^2 = \chi_{K,\beta}\\
  \Rightarrow \widehat{\sigma}_{UB}^2 = \frac{\sum_{i=1}^K n_i(X_i-u)^2}{\chi_{K,\beta}}.
  $$
  这样这里就是95%的可能性方差小于这个UB值。而对于per block cardinality的u的upper bound来说，有下面式子，为u的92.5% upper bound 和 lowe bound
  $$
  \widehat{u}_{UB} = \widehat{u} + 1.96* \widehat{\sigma}_{UB} \\
  \widehat{u}_{UB} = \widehat{u} - 1.96* \widehat{\sigma}_{UB}.
  $$
  这里得出的是92.5%。得出92.5%的可能性来α=0.025和β=0.975值的设置。即：
  $$
  P(u>\widehat{u}_{UB} \lor \sigma>\widehat{\sigma}_{UB}) < P(u>\widehat{u}_{UB}) + P(\sigma>\widehat{\sigma}_{UB}) = 0.075
  $$
  第一个为5%/2，第二个为5%，这样得出0.075，从而得到92.5%。这里计算的时候应该是忽略了概率很小的部分。如果设置为α=0.025和β=0.95，得出的confidence interval为90%。安装Paper中的描述，Paper中第七页中的：We know that with 95% probability, σ^2 is less than σˆ_UB^2 中的95%似乎应该是97.5%???。

* Next Sample Size Calculation。在这一轮采样中，如果下面式子成立，则不需要在继续进行采样了。如果不满足，则需要在第K轮采样使得其满足：
  $$
  \widehat{u} + z_\alpha\frac{\sigma}{\sqrt{N}} \leq (1+\lambda)\widehat{u} \\
  即可以表示为\sigma^2\leq\frac{\lambda^2\widehat{u}N}{z^2}\\
  根据\sigma的推导，有\frac{\sum_{i=1}^K n_i(X_i-u)^2}{\chi_{K,\beta}}\leq\frac{\lambda^2\widehat{u}N}{z^2}\\
  得出N的表达式N = \frac{z^2}{\lambda^2\widehat{u}\chi_{K,\beta}}(\sum_{i=1}^{K-1}n_i(x_i-\widehat{u})+n_k(x_K-\widehat{u})).
  $$
  但是第K次采样的情况在采样之前是不知道的，所以这里使用中心极限定理来进行估计：
  $$
  E[(x_k-\widehat{u})^2] = \frac{\sigma^2}{n_K},\\
  从而将N的估计式重写为: \\
  N=\frac{z^2}{\lambda^2\widehat{u}\chi_{K,\beta}}(\sum_{i=1}^{K-1}n_i(x_i-\widehat{u})+\sigma^2)\\
  这里有可以根据\sigma的计算方式重写为: \\
  N=\frac{z^2}{\lambda^2\widehat{u}\chi_{K,\beta}}(\sum_{i=1}^{K-1}n_i(x_i-\widehat{u}))(1+\frac{1}{\chi_{K-1,\beta}})
  $$
  N为K次采样的总Block数量，第K次可以计算为n=N-N_k-1。这样这里至少需要2轮采样才能使用上面的方法，所以这里会有初始化的操作。

Paper中还总结了几种特殊情况的处理：1. 没有符合条件的rows，这里使用的处理策略是使用上次采样双倍的采样数量，直到找到符合条件的row 或者是到达一个阈值。如果到达了一个阈值还没有发现，则预测为0。2. 对于很复杂的操作，包含多个查询部分的，可以先将其拆解为多个statement，分别执行操作。这里Oracle还引入了SQL Plan Directives来更好的保存统计信息和指导优化器对查询进行优化，具体可以参考[2].

### 0x12 评估

 这里的具体内容可以参看[3].

## 参考

1. Efficient and Scalable Statistics Gathering for Large Databases in Oracle 11g, SIGMOD '08.
2. Distinct Sampling for Highly-Accurate Answers to Distinct Values Queries and Event Reports, VLDB '01.
3. Adaptive Statistics in Oracle 12c, VLDB '17.