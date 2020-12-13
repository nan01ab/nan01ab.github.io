---
layout: page
title: Finding a Needle in Haystack
tags: [Storage, Distributed]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## Finding a Needle in Haystack: Facebook’s photo storage

Haystack是对象存储的经典设计，它是Facebook的照片应用而专门优化定制的对象存储系统。Facebook在峰值时需提供每秒查询超过1 million图片的能力。设计上的关键点：

1. 传统的基于NAS or NFS的存储图片的方式，因为元数据查询而导致了过多的磁盘操作。在Haystack的设计中，追求尽量减少每个图片的元数据，让Haystack能在内存中执行所有的元数据查询。在传统的POSIX的文件系统中，读取一个图片要经过读取目录信息，读取文件信息，最后才能读取文件信息，要经过多次的磁盘操作。访问文件的元数据成为了吞吐量的瓶颈。
2. 每个数据只会写入一次、读操作频繁、从不修改、很少删除；

设计目标：1. 高吞吐量和低延迟。2. 容错。3.  高性价比。4. 简单。

### 架构

  Haystack架构包含3个核心组件：Haytack Store、Haystack Directory和Haystack Cache。

1. Store是持久化存储系统，并负责管理图片的文件系统元数据。Store将数据存储在物理的卷上，此外，不同机器上的多个物理卷对应一个逻辑卷。一个图片存储到一个逻辑卷时，图片被写入到所有对应的物理卷。在物理卷的基础之上加上一个逻辑卷的方法可以解决一个物理卷损坏时导致的数据丢失和服务不可用。
2. Directory维护了逻辑到物理卷的映射以及其的元数据，比如某个图片保存在哪个逻辑卷已经某个逻辑卷的空闲空间的信息。
3.  Cache的功能类似系统内部的CDN，主要处理热门的图片。Cache会尽可能的cache图片数据。Cache的存在也让系统在没有CDN的情况下也能很好的工作。

### 基本操作

访问一个图片：当用户使用Directory为每个图片来构建的一个URL来访问一个图片。这个URL包含几段信息，每一段内容对应了到从浏览器访问CDN(或者Cache)直至最终在一台Store机器上检索到图片的过程。一个典型的URL如下：
 ```
http://<CDN>/<Cache>/<Machine id>/<Logical volume, Photo>
 ```
  第一个部分<CDN>指明了从哪个CDN查询此图片。到CDN后它使用最后部分的URL，包含了逻辑卷和图片ID等信息，即可查找缓存的图片。如果CDN未命中缓存，则从URL中删除<CDN>相关信息，然后访问Cache。Cache的查找过程与之类似，如果还没命中，则去掉<Cache>相关信息，请求被发至指定的Store机器，即<Machine id>。访问的过程也可以不经过CDN，而直接访问Cache。
  Cache向Store请求一个图片时，需要提供逻辑卷id、key、alternate key，和cookie。Cookie的设计主要是为了安全，cookie是个数字，嵌在URL中。当一张新图片被上传，Directory为其随机分配一个cookie值，并作为应用元数据之一存储在Directory。此cookie可以保证所有发往Cache或CDN的请求都是经过Directory“批准”的，而Store，Cache就能放心处理数据。当Store机器接收到Cache机器发来的图片查询请求，由于元数据会被保存在内存之中，可以快速查找。如果图片没有被删除，Store在卷文件（一个卷文件保存了大量的图片）中seek到相应的offset，从磁盘上读取整个图片的数据（这里被称为needle），然后检查cookie和数据完整性，若都合法则将图片数据返回到Cache机器。


  上传一个图片：用户上传一个图片时，图片首先被发送到web服务器。web服务器随后从Directory中请求一个可写逻辑卷，然后web服务器为图片分配一个唯一的ID，web服务器必须向store辑卷id、key、alternate key、cookie和真实数据等信息。Store接受到之后，store创建一个新的needle，添加到卷文件的末尾，更新保存在内存中的元数据和映射。更新一个图片：Haystack的设计是不考虑修改图片的。Store append-only的工作方式也不能很好的支持修改性的操作，又haystack并不允许覆盖needle，所以图片的修改都是直接通过添加一个新needle，其拥有相同的key和alternate key来完成。而如果更新之后的needle被写入到与老的needle不同的逻辑卷，需要Directory更新它的应用元数据，未来的请求都路由到新逻辑卷，如果被写入到同一个逻辑卷，则也会被store保存到同一个物理卷，根据offset的不同就可以判断文件的新旧。删除一个图片：删除图片时，store机器将内存中元数据和卷文件中相应的flag设置为已删除，只是标志已经删除。当接收到已删除图片的查询请求，会被直接返回错误信息。删除操作的最终完成是通过逐个复制needle到一个新的卷文件，并跳过任何重复项、已删除项。

### Store

  读操作只需提供图片ID、哪个逻辑卷、哪台物理Store机器等信息，就可以找到图片，如果机器做不到，则会返回错误信息。每个Store机器管理多个物理卷，而每个物理卷存有百万张图片。一个物理卷是一个非常大的文件（大约为100GB）。文件名字格式如下
```
  /hay/haystack<logical volume id>
```
  所以，store有了logical volume id和图片的offset就能非常快的访问一个图片。Store机器会将其下所有物理卷的文件描述符缓存在内存中。图片ID到文件系统元数据（文件、偏移量、大小等）的映射是检索图片的重要条件，也会全部缓存在内存中。一个物理卷是一个大型的文件，由一个Superblock和一系列的needle组成，每个needle就是一张图片。一个needle会包含以下的字段：

| 字段          | 解释                   |
| ------------- | ---------------------- |
| Header        | 由于恢复的Magic Number |
| Cookie        | 用于安全验证的随机数   |
| Key           | 64bit的Photo ID        |
| Alternate Key | 32bit的Supplemental ID |
| Flags         | 标志，比如删除标准     |
| Size          | 数据长度               |
| Data          | 数据                   |
| Footer        | 由于恢复的Magic Number |
| Data Checksum | 校验值                 |
| Padding       | 需要对齐的填充         |

为了快速的检索needle，Store机器需要为每个卷维护一个内存中的key-value映射。映射的Key就是key和alternate_key的组合，映射的Value就是needle的flag、size、卷offset。如果Store机器崩溃、重启，可以直接分析卷文件来重新构建这个映射。

### Directory

Directory提了供4个主要功能：

1. 提供从逻辑卷到物理卷的映射。上传图片和构建图片URL时都需要使用这个映射；
2. Directory在分配写请求到逻辑卷、分配读请求到物理卷时需保证负载均衡；
3. Directory决定一个图片请求应该被发至CDN还是Cache，这个功能可以让系统动态调整是否依赖CDN；
4. Directory指明那些逻辑卷是只读的。

当系统增加了新机器时，那些新机器是可写的。仅仅可写的机器会收到upload请求，随时间流逝这些机器的可用容量会不断减少，当一个机器达到容量上限，就会标记它为只读。Directory将应用元数据存储在一个冗余复制的数据库，当一个Store机器故障、数据丢失时，Directory在应用元数据中删除对应的项。

### Cache

Cache可理解为一个分布式Hash Table，使用图片ID作为key来定位缓存的数据。Cache并不会缓存所有的图片，缓存的图片要满足以下的信息：

1. 请求来自用户。如果在CDN中没命中，那么这个图片很可能使用不热门的图片，那么也就不太需要缓存。
2. 图片来自一个可以写的store。这个源于经验，热门的图片一般在上传之后很快会被频繁访问，而文件系统不太系统并发的读写。如果没有Cache，可写Store机器往往会遭遇频繁的读请求，还有就是还会主动向cache推送这类型的图片。

### 优化

#### 索引文件

  Store使用索引文件来帮助重启初始化。尽管可以直接通过读取所有的物理卷来重新构建它的内存中映射，但大量数据扫描非常耗时。索引文件允许Store机器快速的构建内存中映射，减少重启时间，索引文件实际就是内存中映射的一个持久保存版本。索引文件的布局和卷文件类似，也是由一个超级块和一系列索引记录组成，每个记录对应到一个needle，索引文件中的记录与卷文件中对应的needle必须保证相同的存储顺序。

| 字段          | 解释                 |
| ------------- | -------------------- |
| Key           | 64bit的Photo ID      |
| Alternate Key | 32bit的Alternate Key |
| Flags         | 当前没有使用         |
| Offset        | 对应needle的偏移     |
| Size          | 对应needle的size     |

  索引文件都是异步更新的，意味着当前索引文件的数据可能不是最新的。当写入一个新图片时，Store同步append一个needle到卷文件末尾，并异步append一个记录到索引文件。删除图片时，Store在对应needle上同步设置flag，但是不会更新索引文件。这些设计是为了避免附加的同步磁盘写，但是也导致了一个needle可能没有对应的索引记录、索引记录中无法得知图片已删除。
  所以在store在重启时，会顺序的检查每个找不到对应索引的needle，重新创建匹配的索引记录，append到索引文件。能快速的识别孤儿是因为索引文件中最后的记录能对应到卷文件中最后的非孤儿needle。由于索引记录中可能无法得知图片已删除，store可能去检索一个实际上已经被删除的图片，所以在检查了内存中的删除之后，还要在读取needle之后再次检查，如果needle中的数据为已经删除，则要更新内存中的信息，并返回错误信息。这里的设计和Bitcask的索引文件设计很像。除此之外，Haystack还通过在线回收已删除的、重复的needle所占据的空间、在内存中只最少量的数据、尽量将相关写操作捆绑批量执行、使用xfs这样对大文件优化的文件系统来优化系统的性能。

##  参考

1. Finding a needle in Haystack: Facebook’s photo storage, OSDI '10.
