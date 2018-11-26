---
layout: page
title: Access Path Selection in a Relational Database Management System
tags: [Database, Optimizer]
excerpt_separator: <!--more-->
typora-root-url: ../
---



##  Access Path Selection in a Relational Database Management System



### 0x00 引言

  这篇Paper是数据库优化器设计中的一篇极为重要的文章，对后面的相关系统的设计产生了很大的影响。这篇Paper要解决的问题就是在用户在不知道数据库具体的实现细节的情况下，数据库的优化器如何讲用户的SQL转化为尽可能高效的具体的查询数据的方式，

```
  SQL statements do not require the user to specify anything about the access path to be used for tuple retrieval. Nor does a user specify in what order joins are to be performed. The System R optimizer chooses both join order and an access path for each table in the SQL statement. Of the many possible choices, the optimizer chooses the one which minimizes “total access cost” for performing the entire statement.
```

.

### 0x01 The Research Storage System

  The Research Storage System (RSS)是System R中负责存储的子系统，它是一个面向元组的系统(tuple-oriented)，这些元组保存的一定大小的page中，不会跨pages存储。为了访问这些元组，这里有两种方式，一种就是顺序扫描,

```
The first type is a segment scan to find all the tuples of a given relation. A series of NEXTs on a segment scan simply examines all pages of the segment which contain tuples, from any relation, and returns those tuples belonging to the given relation.
```

另外一种就是根据索引来获取所需要的元组，

```
These indexes are stored on separate pages from those containing the relation tuples. Indexes are implemented as B-trees, whose leaves are pages containing sets of (key, identifiers of tuples which contain that key). Therefore a series of NEXTs on an index scan does a sequential read along the leaf pages of the index, obtaining the tuple identifi- ers matching a key, and using them to find and return the data tuples to the user in key value order.
```

顺序扫描意味着元组的访问是连续的，而使用索引的方式则不一定，很显然这两种方式适应的是不同的使用环境。这里optimizer一个任务就是决定哪一种方式是更加好的。很显然更加好直观的感受就是跑得更加快，但是这个数据看自己做出决策来说，这个是不能使用的。为了评估一种访问方式的好坏，这里就是使用了COST的概念。



### 0x02 Costs for single relation access paths

  这里先讨论的是在一个relation上面的优化，也就是一个table上面的。这里使用下面这个基本的公式来计算一种access pathde成本：

```
COST = PAGE FETCHES + W * (RSI CALLS)
```

  COST主要是两个部分组成，PAGE FETCHES代表了IO的成本，而RSI CALLS(storage system interface (RSI))代表了CPU的成本。W则是一个比例因子。具体来说，RSI CALLS代表了预测的会返回的元组的数量，在System R中，大部分的CPU实际是化在RSI calls，使用这个就是一个很好的CPU成本的估计。

  一个查询返回的元组的数量与查询中的where中的断言有很大的关系，在这个where的谓词中，每一个具体的谓词会被赋予一个因子F，使用这个因子大概代表会满足这个谓词的元组占所有元组的比例。这样，这里的重点之一就是为了这些因子的估计，这里使用了启发式和基于统计信息的方法，System R会维持下面的一些统计信息:

```
For each relation T,
- NCARD(T), the cardinality of relation T. T中元组的基数。

- TCARD(T), the number of pages in the segment that hold tuples of relation T. segment中的page的数量。

- P(T), the fraction of data pages in the segment that hold tuples of relation T. 
  P(T) = TCARD(T) / (no. of non-empty pages in the segment). 

For each index I on relation T,
- ICARD(I), number of distinct keys in index I. 不同key的数量。
- NINDX(I), the number of pages in index I. 这个索引使用的pages的数量。
```

然后就是一个计算的方式，下面基本就是where中各种谓词的方式如何计算，如果看过《数据库索引设计与优化》这本书的话，就会发现这本书其中的一些内容讲的就是这个，很相似m(._.)m。

选择因子的估计:

```
column = value
  F = 1 / ICARD(column index) if there is an index on column. 
  This assumes an even distribution of tuples among the index key values. F = 1/10 otherwise

column1 = column2
  F = 1/MAX(ICARD(column1 index), ICARD(column2 index)) if there are indexes on both column1 and column2. 
  This assumes that each key value in the index with the smaller cardinality has a matching value in the other index.
  F = 1/ICARD(column-i index) if there is only an index on column-i 
  F = 1/10 otherwise
  
column > value (or any other open-ended comparison)
  F = (high key value - value) / (high key value - low key value)
  Linear interpolation of the value within the range of key values yields F if the column is an arithmetic type and value is known at access path selection time.
  F = 1/3 otherwise (i.e. column not arithmetic)
  There is no significance to this number, other than the fact that it is less selec- tive than the guesses for equal predicates for which there are no indexes, and that it is less than 1/2. We hypothesize that few queries use predicates that are satis- fied by more than half the tuples.
  
column BETWEEN value1 AND value2
  F = (value2 - value1) / (high key value - low key value)
  A ratio of the BETWEEN value range to the entire key value range is used as the selectivity factor if column is arithmetic and both value1 and value2 are known at access path selection.
  F = 1/4 otherwise
  Again there is no significance to this choice except that it is between the default selectivity factors for an equal predicate and a range predicate.
  
column IN (list of values)
   F = (number of items in list) * (selectivity factor for column = value) This is allowed to be no more than 1/2.
   
columnA IN subquery
  F = (expected cardinality of the subquery result) / (product of the cardinalities of all the relations in the subquery’s FROM-list).
  The computation of query cardinality will be discussed below. This formula is derived by the following argument: Consider the simplest case, where subquery is of the form “SELECT columnB FROM relationC ...”. Assume that the set of all columnB values in relationC contains the set of all columnA values. If all the tuples of relationC are selected by the subquery, then the predicate is always TRUE and F = 1. If the tuples of the subquery are restricted by a selectivity factor F’, then assume that the set of unique values in the subquery result that match columnA values is proportionately restricted, i.e. the selectivity factor for the predicate should be F’. F’ is the product of all the sub- query’s selectivity factors, namely (subquery cardinality) / (cardinality of all pos- sible subquery answers). With a little optimism, we can extend this reasoning to include subqueries which are joins and subqueries in which columnB is replaced by an arithmetic expression involving column names. This leads to the formula given above.
  
(pred expression1) OR (pred expression2)
  F = F(pred1) + F(pred2) - F(pred1) * F(pred2)
  
(pred1) AND (pred2)
   F = F(pred1) * F(pred2)
  Note that this assumes that column values are independent.
  
NOT pred
F = 1 - F(pred)
```

计算公式:

```
Unique index matching an equal predicate -> 1+1+W

Clustered index I matching one or more boolean factors -> F(preds) * (NINDX(I) + TCARD) + W * RSICARD

Non-clustered index I matching one or more boolean factors -> F(preds) * (NINDX(I) + NCARD) + W * RSICARD or F(preds) * (NINDX(I) + TCARD) + W * RSICARD if this number fits in the System R buffer

Clustered index I not matching any boolean factors -> (NINDX(I) + TCARD) + W * RSICARD

Non-clustered index I not matching any boolean factors -> (NINDX(I) + NCARD) + W * RSICARD
or (NINDX(I) + TCARD) + W * RSICARD if this number fits in the System R buffer

Segment scan -> TCARD/P + W * RSICARD
```

以上面的顺序扫描和索引扫描为例，根据上面的统计信息、因子和计算公式，就可以计算出两种访问方式花费的COST，从而可以从中选择出更加"好"的访问方式。当然只考虑这些是不够的，这里还要考虑的是order by和group by，如果存在下面所说的interesting order，就必须将这里的COST页考虑进去，

```
Using an index access path or sorting tuples produces tuples in the index value or sort key order. We say that a tuple order is an interesting order if that order is one specified by the query block’s GROUP BY or ORDER BY clauses.
```

.

### 0x03 Access path selection for joins

  emmmm，关于join在数据库中的优化是一个很大的话题。在这篇Paper中也直讲了一些基本的东西，这离总结也会只总结最基本的几个东西。Join的优化两个重要的的问题就是1. 选择怎么样的join算法， Paper中讲了 nested-loop based joins和merging-scan based joins，当然现在hash join也是一个很常用的方法。另外一个更加复杂的问题就是如何选择join的顺序了，这个问题被研究了几十年了。为了减少考虑的join顺序，也使用了一些启发式的方法，

```
A heuristic is used to reduce the join order permutations which are considered. When possible, the search is reduced by consideration only of join orders which have join predicates relating the inner relation to the other relations already participating in the join. This means that in joining relations t1,t2,...,tn only those orderings til,ti2,...,tin are examined in which for all j (j=2,...,n) either

(1) tij has at least one join predicate with some relation tik, where k < j, or

(2) for all k > j, tik has no join predicate with til,tit,...,or ti(j-1).
```

为了找到更优的join方式，这里使用构造一棵join搜索树的方式，这个方式在后面的关于这个的研究中也是很常用的。搜索树是通过对join的relation进行迭代来构造，先是找到一个访问一个relation的方式，接下来, 找到了与这些关系join的最佳方式。更大规模的join操作可以看作是小的join组成的，然后通过这些构造出这棵树。Paper中给出了几个例子，可以参看[1].

```
 For each plan to join a set of relations, the order of the composite result is kept in the tree. This allows consideration of a marge scan join which would not require sorting the composite. After the complete solutions (all of the relations joined together) have been found, the optimizer chooses the cheapest solution which gives the required order, if any was specified.
```

join操作也会设计到成本的计算，它以前面的单个relation上面的查询为基础。设C-outer(path1)为outer的relation根据前面的方式计算出来的COST，N为满足谓词的外部relation的元组的基数，N的计算方式如下：

```
N = (product of the cardinalities of all relations T of the join so far) * (product of the selectivity factors of all applicable predicates).
```

 C-inner(path2)则代表内部relation中访问的计算出的成本，这样对于nested-loop-join，这个join操作的成本的计算方式如下：

```
C-nested-loop-join(pathl,path2)= C-outer(path1) + N * C-inner(path2)
```

对于merge-join操作，它的成本来自merge操作的和可能需要进行排序的成本，计算的公式是：

```
C-merge(pathl,path2)= C-outer(path1) + N * C-inner(path2)
```

 对于内部relation为排序为临时relation的情况下，内部扫描就像段扫描, 只是合并扫描方法利用了内部relation排序的事实, 因此不需要扫描整个内部relation以查找匹配。这里如果对merge-join有些了解就比较容易立即了。对于这种情况, 使用以下公式来计算内部扫描的成本

```
C-inner(sorted list) = TEMPPAGES/N + W*RSICARD

where TEMPPAGES is the number of pages required to hold the inner relation. This formula assumes that during the merge each page of the inner relation is fetched once.
```

 可以发现这里的计算方式竟然是一样的。当然这里还是存在区别的，对于merge-join来说，有时候内部扫描的成本会低很多。这里对于join算法就放在之后的一些总结中讨论吧。



### 0x04 Nested Queries

 这里将的就是子查询（嵌套查询），比如下面的例子，

```sql
SELECT NAME
FROM EMPLOYEE
WHERE SALARY = 
    (SELECT AVG(SALARY)
     FROM EMPLOYEE);
     
# or

SELECT NAME
FROM EMPLOYEE
WHERE DEPARTMENT_NUMBER IN
(SELECT DEPARTMENT_NUMBER FROM DEPARTMENT
WHERE LOCATION='DENVER')
```

上面是两种不同的例子，一种是返回一个值的，就如上面的第一个，而第二个则是返回一组值的例子。对于这样的例子，都是先对自查询进行优化，然后将返回的结果做为high level查询的一部分。下面是另外的一种查找的例子，

```sql
SELECT NAME
FROM EMPLOYEE X
WHERE SALARY > (SELECT SALARY FROM EMPLOYEE WHERE EMPLOYEE_NUMBER= X.MANAGER)
```

 这样的叫做相关子查询，它的处理方式更加复杂。外部查询执行一次内部就是执行一次，这里要关心的是这样对成本的影响。

```
A correlation subquery must in principle be re-evaluated for each candidate tuple from the referenced query block. This re-evaluation must be done before the correlation subquery’s parent predicate in the higher level block can be tested for acceptance or rejection of the candidate tuple.
```

当然还会有更加复杂的查询。



### 0x05 总结

 数据库优化器的基本思想，在今天看来这些东西都是很基本的东西，却在数据库的发展历程中起到了至关重要的作用。



## 参考

1. Access Path Selection in a Relational Database Management System, SIGMOD 1979.