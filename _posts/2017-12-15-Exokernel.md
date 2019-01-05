---
layout: page
title: Exokernel -- An Operating System Architecture Application-Level Resource Management
tags: [Operating System]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## Exokernel: An Operating System Architecture for Application-Level Resource Management 

### 引言

 Exokernel是一种Kernel的架构，Exokernel的Paper发表在SOSP‘95上面。它最大的特点就是将一些资源的管理可以交给应用程序完成，

```
Measurements show that application-level virtual memory and interprocess communication primitives are five to 40 times faster than Ultrix’s kernel primitives. Compared to state-of-the-art implementations from the literature, the prototype exokernel system is at least five times faster on operations such as exception dispatching and interprocess communication.
```

.

### 基本思路

   Exokernel使用的其实是一种很激进的思路。Exokernel认为传统的统一抽象的方式并不能最佳地发挥出应用的性能。OS来管理资源的一个缺点就是它不知道如果抽象、管理资源才是最适应某一类应用的，对这个最了解的一应该就是应用本身。Exokernel的做法就是Kernel只复杂处理一些基本的管理、安全相关的问题，而对于资源具体的管理，就通过接口暴露给应用自己去管理。Exokernel要实现它的功能，主要要解决的问题有以下这些：

```
(1) tracking ownership of resources, 
(2) ensuring protection by guarding all resource usage or binding points, and 
(3) revoking access to resources.
```

  为了解决这些问题，Exokernel使用了secure bindings技术来安全地绑定硬件资源，使用visible revocation来取消一个应用对资源地使用，通过abort protocol来强制取消不合作的应用。此外，Exokernel的设计中体现了下面的一些的设计原则：

```
* Securely expose hardware;
* Expose allocation;
* Expose Names;
* Expose Revocation;
```

.

#### Secure Bindings

 在讲资源暴露给应用管理之后，Exokernel一个最重要的任务就是保护应用不受另外的应用的影响。应用彼此之间都是假设为不能信任的。这里其实就是如何安全地资源多路复用。为了高性能地实现Secure Bindings，Exokernel使用了如下的方式：1. 保护检查必须可以被实现为kernel或者硬件可以高效执行的操作；2. 一个Secure Bindings执行权限验证的时候只在Bind的时候进行。

```
 Application-level software is responsible for many resources with complex semantics (e.g., network connections). By isolating the need to understand these semantics to bind time, the kernel can efficiently implement access checks at access time without understanding them. Simply put, a secure binding allows the kernel to protect resources without understanding them.
```

  这里使用3中基本的技术来实现Scure Bindings：

* 硬件机制(hardware mechanisms)，如果硬件直接支持修改的安全检查机制的话，这里的效率就是最理想的。比如内存页面访问时候的权限检查；
* 软件缓存(software caching)，Secure bindings的信息也可以缓存在内核中；
* 应用代码下放(downloading application code)，这种方式就是将应用的代码下放在内核中运行；



#### Visible Resource Revocation

  资源管理的权限交给一个应用之后，必须有方法在合适的时候回收这些资源。一般的资源回收都是应用可感知的，当然也可以在应用不知情的情况下就将资源回收了。在应用可知的情况下进行资源回收可以让应用做一些状态保存的善后的工作。



#### Abort Protocol

 Exokernel必须可以在应用没有相应资源回收的情况下强制地回收资源。Exokernel采取的策略可以是“先礼后兵“，开始可以看作是“请把某某资源归还“，如何应用没有响应，可以变成“命你在50ms内归还某某资源“。对此此类会有一个abort的协议。这个协议可以就是简单的kill不遵循“命令”的应用。但是Paper中认为这种方式存在明显的确定，因而采用的是解除Secure Binding的方式，

```
To record the forced loss of a resource, we use a repossession vector. When an exokernel takes a resource from a library operating system, this fact is registered in the vector and the library operating system receives a “repossession” exception so that it can update any mappings that use the resource.
```

.

## Aegis

  Paper中描述了一个Exokernel架构的一个实现。并在多种环境对多种的应用进行了测试，从Paper中的结果来看，Aegis的编写很优秀。



## 参考

1. Exokernel: An Operating System Architecture for Application-Level Resource Management, SOSP 1995.

