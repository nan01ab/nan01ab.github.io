---
layout: page
title: Consistent Hash
tags: [Algorithm]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## Consistent Hash

### About

  一致性hash算法的概念在很多地方都可以找到很详细的说明，这里就不在说明。简单的来说：一致性hash就是如何将N个对象就可能平均分配到M个节点之中（这里抽象为一个节点），并且在节点的数量在增加或者减少时尽量减少对象的重新分配，还要保证在节点增加时原有已分配的内容可以被映射到新的缓冲中去，而不会被映射到旧的缓冲集合中的其他缓冲区。

### 基于环的一致性Hash算法

#### 基本思想

  这个算法时最经典的一致性Hash算法。这个算法的基本思想是将value映射为一个值，通常情况下是一个int，而hash值存在一个数值空间，将这个空间抽象为一个环，比如0 - 2^ 32-1，0和2^32-1首尾相接。在这个空间之中，从对象的hash值出发（沿着一个方向），直到遇到一个节点，那么这个对象就放入这个节点。当节点减少时，只有这个节点到下一个节点之间的对象需要重新移动；当在环中增加一个节点时，只有这个节点到上一个节点之间的对象需要重新移动。满足了一致性hash的基本要求。

#### 一个优化

  Hash算法不能保证完全hash值完全平均分配，而且完全平均分配从理论上证明了这是不可能的（具体记不清了）。不过在节点足够多以及hash函数很好的话，这还好。但是当节点的数量很少时，这个就很容易发生不平均的问题。为了优化这种情况，引入了“虚拟节点”的概念：
```
所谓虚拟节点，就是一个实际的节点回应了对个虚拟的节点。
```
  通过虚拟出更多的节点来优化这个问题。

### Jump Consistent Hash

```
jump consistent hash, a fast, minimal memory, consistent hash algorithm that can be expressed in about 5 lines of code. In comparison to the algorithm of Karger et al., jump consistent hash requires no storage, is faster, and does a better job of evenly dividing the key space among the buckets and of evenly dividing the workload when the number of buckets changes. Its main limitation is that the buckets must be numbered sequentially, which makes it more suitable for data storage applications than for distributed web caching.
```

  第一次看到这个算法时，就觉得非常惊艳，首先就是基本原理与经典的算法完全不一样，然后就是算法实现的简洁（这个算法的描述之中，上文的节点描述为桶，这一节的代码没有明确说明的话都来自论文[2])，这里一开始看上去有点不知所以然:
```c
int32_t JumpConsistentHash(uint64_t key, int32_t num_buckets) {
    int64_t b = -1, j = 0;
    while (j < num_buckets) {
        b = j;
        key = key * 2862933555777941757ULL + 1;
        j = (b + 1) * (double(1LL << 31) / double((key >> 33) + 1));
    }
    return b;
}
```

先来看看jump consistent hash的设计目标是：

```
1. about the same number of keys map to each bucket
2. the mapping from key to bucket is perturbed as little as possible when the number of buckets is changed.
Thus, the only data that needs to move when the number of buckets changes is the data for the relatively small number of keys whose bucket assignment changed.
```
​    这个与通常的一致性hash算法没有什么不同，而jump consistent hash的设计思路是：**计算当bucket数量变化时，有哪些输出需要变化**。论文中介绍这个算法采用了循序渐进的方法，一个基本的规律是：num_buckets从n变化到n+1后，ch(k,n+1) 的结果中，应该有占比 n/(n+1) 的结果保持不变，而有 1/(n+1) 跳变为 n+1。
```
 In general, ch(k, n+1) has to stay the same as ch(k, n) for n/(n+1) of the keys, and jump to n for the other 1/(n+1) of the keys.
```
  这个也是这个算法被称为jump一致性hash的原因。基于这个基本的思想，有了第一个算法的版本：
```c++
int ch(int key, int num_buckets) {
    random.seed(key) ;
    int b = 0; // This will track ch(key, j +1) .
    for (int j = 1; j < num_buckets; j ++) {
        if (random.next() < 1.0/(j+1) ) b = j ;
    }
    return b;
}
// 这个的实现的就是对于每一个桶，有 1/j+1的可能jump。返回最终jump的桶的编号。
// 复杂度为O(n)，
```

   如何优化？可以观察到大多数情况下b=j 是不会执行的，这是一个优化的切入点：
```
To develop the algorithm, we will treat ch(key, j) as a random variable, so that we can use the notation for random variables to analyze the fractions of keys for which various propositions are true. That will lead us to a closed form expression for a pseudo-random variable whose value gives the destination of the next jump.
```
   这个优化的基本思想就是直接获取求出下一跳。这里记上一个跳变结果是b，假设下一个结果以一定概率是 j ，那么从b+1到j-1，这中间的多次增加桶都不能跳变,于是就有(if only if，iff) ch(k, i) = ch(k, b+1)，即桶从b+1到i的过程之中，k一直对应了同一个桶（i是中间的一个值），这里的j就满足：
```
P(j ≥ i) = P(ch(k, i) = ch(k, b+1))
ch(k,i)==ch(k,b+1) 即从b+1到i的过程中，连续多次增加桶的时候都没有跳变。
```

也就是说j大于等于i的概率等于中间b+1到中间i一只不跳变的概率，而一次不变的概率为，也就是i增加到i+1的时候，对象k还在同一个桶里面的概率是：
```
P( ch(k,i)==ch(k,i+1) ) = i/(i+1)

这样就有:
Notice that since P( ch(k, 10) = ch(k, 11) ) is 10/11, and P( ch(k, 11) = ch(k, 12) ) is 11/12, then P( ch(k, 10) = ch(k, 12) ) is 10/11 * 11/12 = 10/12. In general, if n ≥ m, P( ch(k, n) = ch(k, m) ) = m / n. Thus for any i > b,
```

 根据这里就有，一个递推就可以得到多次不变的概率就是：

```
P(j ≥ i) = P( ch(k, i) = ch(k, b+1) ) = (b+1) / i
```

现在，取一个在[0,1]区间均匀分布的随机数r，因为我们想得到P(j ≥ i) = (b+1) / i , 就当且仅当 r ≤  (b+1) / i时，有 j ≥ i，转换一下就是有 i ≤  (b+1) / r . 因为 j的下界是i，所以j得大于等于i才行，又有i ≤  (b+1) / r，加上它们都得事整数，就得到 j = floor( (b+1) /r)。

```
Now, we generate a pseudo-random variable, r, (depending on k and j) that is uniformly distributed between 0 and 1. Since we want P(j ≥ i) = (b+1) / i, we set P(j ≥ i) iff r ≤ (b+1) / i. Solving the inequality for i yields P(j ≥ i) iff i ≤ (b+1) / r. Since i is a lower bound on j, j will equal the largest i for which P(j ≥ i), thus the largest i satisfying i ≤ (b+1) / r. Thus, by the definition of the floor function, j = floor((b+1) / r).
```

得到最终的算法代码:

```c++
int ch(int key, int num_buckets) {
    random. seed(key) ;
    int b = -1; //  bucket number before the previous jump
    int j = 0; // bucket number before the current jump
    while(j < num_buckets) {
        b=j;
        double r = random.next(); //  0<r<1.0
        j = floor( (b+1) /r);
    }
    return b;
}
// 时间复杂度为O(log(n))
// 论文的最终的实现就是这个版本的一个实现。
```

  根据论文中的数据，这个算法这方面都胜出一筹。 Google guava里面的实现(comm.hash.Hashing)，使用了第一个实现，并没有使用最优的算法。

```java
  public static int consistentHash(long input, int buckets) {
    checkArgument(buckets > 0, "buckets must be positive: %s", buckets);
    LinearCongruentialGenerator generator = new LinearCongruentialGenerator(input);
    int candidate = 0;
    int next;

    // Jump from bucket to bucket until we go out of range
    while (true) {
      next = (int) ((candidate + 1) / generator.nextDouble());
      if (next >= 0 && next < buckets) {
        candidate = next;
      } else {
        return candidate;
      }
    }
  }
```

.

### 有界负载一致性Hash算法

#### 基本思想

  这个算法同样是Google发明的一个算法。算法的基本思想是：
```
每台服务器都有一个最大负载容量限制，算法希望容量能接近于平均负载。为了实现分配的均匀，对于一个参数ε，该算法将每个箱子的容量上下界设置为平均负载的上下（1 +ε）倍，在分配时，每个对象（论文中称为ball）进入顺时针方向中第一个没有满的桶之中（论文中称为bin）。
```

   从这个基本思想可以看出，算法想通过一个设定一个容量的方法来改善分配不平均的现象。一致性hash第一个满足了，哪第二点呢？ 在解决这个问题之前，先来看这几个结论：
```
1. No ball passes a non-full bin。
```

 Ball passes是指本来应该分配至A的对象，由于A的容量已满，被分配到了B。这个结论就是，对象会被分配到它遇到了第一个没有满的bin之中。所以查找一个对象的方法顺时针查找每一个bin，直到找到对象或者遇到一个没有满的bin，但是没有找到这个对象。当插入一个对象的时候，当插入的就是第一个bin的时候，称为简单插入；当遇到的第一个bin满时，为了更加灵活，算法使用了一个优化的方法：不是将当前的对象移入下一个没有满的bin之中，而是可以将一定数量的对象移入顺时针方向其它的bin之中。为了仍然满足1，将这个看作full（文中有overfull，full和non-full）。之前的方法只能到达最近的non-full的bin，现在可以移动选择的一个对象，算法在所有的球上假设一个线性的顺序, 这是独立于它们的插入顺序。

```
Temporarily, we allow a bin to be overfull in the sense that the load exceeds the capacity. When a ball arrives, it is first placed in the bin it hashes to, which may become overfull. If the system has an overfull bin, we forward any of its balls to the next bin.
```

  当删除一个bin时，就是将它的容量看着0。

```
Removing a bin has the same effect as setting its capacity to 0. It is now overfull, and then we forward balls from overfull bins arbitrarily until no bin is overfull. Invariant 1 was never violated.
```

  当删除一个对象时，为了满足1，可能必须要找一个对象来替代它，删除一个对象之后，bin中会产生一个空洞（论文中被称为hole），回填这个hole时，是扫描这个bin之后所有满的bin。可能会产生连锁反应，为了回填这个hole，造成另外的hole，又要回填，这只是初级的方法，论文后面有更优化的算法。添加一个bin时，算法会尽可能的移动对象来满足1。添加一个bin的时候，就看着事造成了这个bin容量的这么多的hole，然后按照前面的方法处理。

```
When a bin is added, it starts with as as many holes as its capacity. We then keep filling holes as long as possible so Invariant 1 is restored. No capacity constraint is violated in this process.
```

 论文之后有花了很长的篇幅来说明该算法更多的内容，包括优化的删除对象的方法已经容量的动态调整。

### CRUSH

  这篇文章介绍的算法是一个比一个难的。这个CRUSH是在ceph里面使用的算法。这个算法包含了更加多的内容，在此之前可能需要去读一读ceph这里相关的文章。

## 参考

1. http://arxiv.org/ftp/arxiv/papers/1406/1406.2294.pdf, jump consistent hash.
2. https://arxiv.org/pdf/1608.01350.pdf, Consistent Hashing with Bounded Loads.