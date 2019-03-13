---
layout: page
title: NetVM and OpenNetVM
tags: [Network, Virtualization]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## NetVM: High Performance and Flexible Networking using Virtualization on Commodity Platforms
### 0x00 引言

   NetVM是一种网络虚拟化的解决方案，它建立在KVM的虚拟化平台和Intel的DPDK。在NetVM之前，网络虚拟化的解决方案有这样的一些，1. 完全虚拟化的方案，使用软件来模拟网卡的行为，这样缺点很明显，就是性能相对来说很低，2. 直接访问的方案，这个的缺点就是同一时间只能被一个VM使用，3. 另外一个就是intel提出的SR-IOV的方法，通过虚拟多个VF的方法来满足被多个VMs使用，但是NetVM认为它的缺点是灵活性不足，比如无法实时迁移，支持的VF的数量也是有限的。针对之前的系统存在的这些问题，这里NetVM就提出了它的解决方案，

```
...  Our evaluation shows how NetVM can compose complex network functionality from multiple pipelined VMs and still obtain throughputs up to 10 Gbps, an improvement of more than 250% compared to existing techniques that use SR-IOV for virtualized networking.
```

![netvm-compare](/assets/img/netvm-compare.png)

### 0x01 基本架构

  在[3]中总结了一些这样的系统使用的一些性能优化的技术，比如，1. 轮询取代中断，2. Zero-Copy技术，3. 使用一些高效的虚拟化的技术，4. 网络协议栈的优化。由于NetVM的定位，这里使用前面三种的方式多一些。NetVM的基本架构如下图所示。NetVM虚拟化是基于的KVM，一种Linux内核支持的半虚拟化的技术，网络处理的框架利用了DPDK，DPDK有实现前面一些的优化技术。DPDK是支持Zero-Copy的，NetVM在此基础了为了支持到虚拟机中的Zero-Copy，这里使用的是基于共享内存的方式。具体而言，这里使用了两个部分，第一个部分是小一点的，一个共享内存区域(宿主机和每个虚拟机共享)，用于传输Packet的描述符，另外一个是使用大内存页的Huge Page区域(可以宿主机和一组的虚拟机共享)。所以这样的工作方式决定了NetVM只适合用于VM可信的环境中，在下图中可见，如果是不可信的VM，则使用原来的方案。

![netvm-arch](/assets/img/netvm-arch.png)

  NetVM借助DPDK的功能，可以以轮询的方式直接让网卡读数据包Huge Page中。NetVM接下来的重要功能就是根据包的Header，甚至是包的内容，也或者是每个VM不同的负载的信息来转发这些数据包。NetVM将包的描述信息添加到基于共享内存的ring buffer中，VM即可依据这些信息从Huge Page区域中读取对应的数据。这样就实现了Zero-Copy，

```
... Forwarding simply repeats the above process -- NetVM copies the descriptor into the ring buffer of a different VM so that it can be processed again; the packet data remains in place in the huge page area and never needs to be copied (although it can be independently modified by the guest applications if desired).
```

 为了优化这里通信的性能，NetVM通过Lockless的技术消除了一些锁的使用，关于环形队列之类的结构的无锁设计有不少的参考。另外，这里利用了RSS技术来提高性能。除此之外，NUMA环境下面的一些优化也是NetVM考虑的，NetVM在分配内存的时候，会将请求的内存分为n份(即Sockets)的数量，分配多各个Socket上面。然后Hypervisor会创建系统数量的线程来处理接收/发送请求。这样每个线程只会处理自己所在Socket上面的数据。

![netvm-huge-page](/assets/img/netvm-huge-page.png)

  分拆内存区域的方法虽然解决了NUMA环境下远程内存访问的问题，不过导致的另外一个问题就是这些内存是不连续的。NetVM会将这些内存映射到一个在VM看起来连续的区域。另外，这里NetVM通过一些位运算的方式来快速确定一个数据包应该被放到另外一个巨页，

```
... an index map that converts a packet address to a huge page index. The index is taken from the upper 8 bits of its address (31st bit to 38th bit). The first 30 bits are the offset in the corresponding huge page, and the rest of the bits (left of the 38th bit) can be ignored. We denote this function as IDMAP(h) = (h >> 30)&0xFF, where h is a memory address. This value is then used as an index into an array HMAP[i] to determine the huge page number.
```

### 0x02 实现

NetVM的具体实现上，主要包含了这样的几个组件，如下图所示，

![netvm-impl](/assets/img/netvm-impl.png)

* NetVM Manager，运行在Hypervisor之中，通过一个Socket于QEMU通信。启动一个VM的时候，会请求NetVM初始化相关的数据结构和内存区域。另外，它还会管理VM的信任级别。

* NetVM Core Engine， NetVM Core负责初始化如CPU核心映射之类的用户设置、NIC端口以及队列配置等。分配Huge Page区域。基于DPDK来处理数据包等任务。

* Emulated PCI，KVM不能实现宿主机和VMs之间直接的共享内存，NetVM这里的解决方式是通过模拟PCIe来解决，这样实际的访问就可以NetVM来处理，

  ```
  ... NetVM needs two seperate memory regions: a private shared memory (the address of which is stored in the device’s BAR#0 register) and huge page shared memory (BAR#1). The private shared memory is used as ring buffers to deliver the status of user applications (VM → hypervisor) and packet descriptors (bidirectional). Each VM has this individual private shared memory.
  ```

* NetLib and User Applications，即提供给用户的接口。

### 0x03 评估

  Paper中基于NetVM实现了一个L3 Forwarder，一个Click发给的用户空间路由器，一个防火墙等，它们表现处理的性能都很好看。这里的具体信息可以参看[1],

## OpenNetVM: A Platform for High Performance Network Service Chains

### 0x10 引言

  OpenNetVM可以看作是NetVM一个开源的实现，强调比其它的包处理框架更高层次的抽象(具体是??)。之前看过的mTCP的开源带来中也有适配OpenNetVM的部分。基本上使用了NetVM的架构，不过OpenNetVM面向的是Container的环境，

```
 Our evaluation achieves throughputs of 68 Gbps when load balancing across two NF replicas, and 40 Gbps when traversing a chain of five NFs, realizing the potential of deploying software services in production networks.
```

### 0x11 基本架构

 如前面所言，OpenNetVM基本上还是使用了NetVM的架构，

* NF Manager，顾名思义就是负责各种管理的工作。比如在OpenNetVM初始化的时候，它负责使用DPDK提供的一些功能来初始化内存。这里分配内存的策略也是基于NetVM中使用的策略。OpenNetVM不同于利用KVM实现的方式，它可以直接使用虚拟地址的方式。这个相比于NetVM还有模拟PCIe的方式简单很多，

  ```
  ... When NFs start, they find the shared memory address in a configuration file and then map the memory regions using the same base virtual address as the manager. This ensures that a packet descriptor containing a virtual address of where the data is stored can be correctly interpreted by both the manager and the NFs without any translation. 
  ```

  初次之外，NF Manager还要处理NFs启动、关闭以及一些相关的数据结构等。还有就是流的管理。流的管理主要设计到的Flow Director 和 Flow Tables。OpenNetVM这里定义了Service Chains，即定义在一个流上的一组的行为(Actions)，这些行为可以理解为就是一个NF，它会用一个Service ID来表明。Flow Director 和 Flow Tables 就是设计到这些Actions和Flows之间的处理。

* Network Functions，使用NFLib提供的API实现程序的一些功能。

![opennetvm-arch](/assets/img/opennetvm-arch.png)  

### 0x12 评估

  这里的具体信息看参看[2],

![opennetvm-perf](/assets/img/opennetvm-perf.png)

## 参考

1. NetVM: High Performance and Flexible Networking using Virtualization on Commodity Platforms, NSDI '14.
2. OpenNetVM: A Platform for High Performance Network Service Chains, HotMIddlebox '16.
3. NFV 数据平面的网络性能优化技术, 2017.