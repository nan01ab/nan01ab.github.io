---
layout: page
title: Linux BtrFS
tags: [File System]
excerpt_separator: <!--more-->
typora-root-url: ../
---



### BTRFS: The Linux B-Tree Filesystem 



### 0x00 引言

  BtrFS号称是Linux的下一代文件系统，不过感觉现在进展不怎么样，目前性能也渣。虽然有一大堆的新功能和高级特性，不过bug一大堆。从2007年开始开发到现在都11年了，还是这个鬼样子，有种要扶不上墙的感觉。这篇Paper就介绍了BtrFS的基本技术:



### 0x01 BtrFS特点

  诸多的新功能:

```
(1) CRCs maintained for all metadata and data;
(2) efficient writeable snapshots, clones as first class citizens; 
(3) multidevice support;
(4) online resize and defragmentation;
(5) compression;
(6) efficient storage for small files;
(7) SSD optimizations and TRIM support.
```

.

挖坑，先去吃烧烤，之后补上



.

## 参考

1. Rodeh, O., Bacik, J., and Mason, C. 2013. BTRFS: The Linux B-tree filesystem. ACM Trans. Storage 9, 3, Article 9 (August 2013), 32 pages.  DOI:http://dx.doi.org/10.1145/2501620.2501623 