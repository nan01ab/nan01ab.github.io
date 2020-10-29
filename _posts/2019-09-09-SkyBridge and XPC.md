---
layout: page
title: SkyBridge and XPC, Secure and Efficient Inter-Process Communication
tags: [New Hardware, Virtualization, Operating System]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## SkyBridge: Fast and Secure Inter-Process Communication for Microkernels

### 0x00 核心思路

 这篇Paper是上交一篇优化微内核的IPC性能的一篇Paper。SkyBridge的思路是利用x86处理器的一些虚拟化的功能来实现高性能的IPC，同时还能实现传统的不同的进程隔离在不同的地址空间之内。Paper中先分析了IPC的性能开心的来源，主要有这样的几个部分：1. Mode Switch，即内核态和用户态的转变， 比如涉及到的SYSCALL, SWAPGS 和 SYSRET指令一般的开心分别是82, 26 和 75左右；2，微内核的情况下，IPC的client和server一般在不同的地址空间之内，这样就涉及到一个地址空间的切换，这里的性能开销在开启PCID (process ID)功能的情况下在186 cycles，如果使用了一些缓解Meltdown attack攻击的方法，开心上升到372 cycles；3. 另外的一些事Software IPC logic的开销。以上的几个部分是直接的开心，另外的几个部分是间接的开心，比如flush TLB和cache带来的开销。intel提供的VMFUNC指令可以在no-root的内核和用户模式下调用VM functions，这些由hypervisor管理，比如可以用来处理GPA和HPA之间的转化，

```
  EPTP (the pointer to an EPT) switching is one of these VM functions, which allows the guest to load a new value for the EPTP from an EPTP list stored in the Virtual Machine Control Structure (VMCS) configured by the hypervisor. The new EPT then translates subsequent guest physical addresses (GPA) to host physical addresses (HPA). The EPTP list can hold at most 512 EPTP entries. T
```

 VMFUNC的性能也是比较高的，在VPID功能开启的情况下，TLB不需要flush，这样的开心甚至低于write CR3的开销，

```
Instruction or Operation     Cycles
write to CR3                 186±10
no-op system call w/ KPTI    431±13
no-op system call w/o KPTI   181±5
VMFUNC                       134±3
```

SkyBridge就是利用这个VMFUNC和EPT的功能，在执行IPC操作的时候，client执行的操作切换到servicr在的地址空间执行。SkyBridge的server提供IPC的服务的时候，server需要提供给kernel的IPC的function的地址和最大的能接受的并发请求的数量，kernel会映射trampoline-related code 和 data到server的地址空间之内，并会返回一个server ID。Client要需要向kernel主要，要提供server ID，也会映射trampoline- related code 和 data pages到client的地址空间。Kernel会为client和对应的seerver创建一个EPT。Client通过direct_server_call调用server的函数的时候，trampoline将client的状态信息保存到stack中，然后调用VMFUNC，切换EPT。切换之后，后面的地址转换就是由server的page table来完成的。之后trampoline install server的stack，完成对应的函数调用。

### 0x02 基本设计

在这样的设计下面，SkyBridge主要完成这样的几个工作，

* Efficient Virtualization。Skeybridge的Kernel分为两层，一层是rootkernel，一层是subkernel。Rootkernel作为一个hypervisor的功能，这里SkyBridge将其做的就可能的简化，并且配置大部分的操作都不会触发VM Exits，即subkernel退出到rootkernel。Rootkernel管理EPT，使用1GB的page来提高性能。

  ```
  In our current implementation, the Rootkernel contains han- dlers for CPUID instructions, VMCALL instructions and EPT violations. The VMCALL instruction is leveraged by the Rootkernel to implement an interface to communicate with the Subkernel.
  ```

  Subkernel需要管理client和server的一些行为，比如处理注册的问题，将trampoline code pages和stack pages映射到server and/or client的地址空间，为server分配server ID。要使用SkyBridge IPC的时候，subkernel需要请求rootkenel将其在EPT level进行绑定，rootkernel处理这个的时候，会copy一个new server EPT，并映射client的page table到server的page table，然后添加一个新的EPT到client的EPTP list。Subkernel在决定将一个进程switch到另外一个进程的时候，需要通知rootkernel来install下一个进程的 EPTP list。对于client和server之间的数据传输，数据量较少的时候直接使用寄存器来进行传输，这个也是一些IPC使用的优化方式。对于比较大的数据，则放入到一个shared buffers中，buffer的数据根据注册的时候需要的数量来进程创建。

  这里还提到一个Process Misidentification问题的处理，sender陷入内核的时候，这个时候实际上在receiver的地址空间里面执行。当其invoke micro- kernel’s services as a receiver的时候，可能造成将这个识别为原来进程的问题，这里将其称之为Process Misidentification。SkyBridge的解决方式是在一个identity page 记录下进程的一些信息。

![](/assets/png/skybridge-arch.png)

* Lightweight Virtual Address Switch。SkyBridge通过VMFUNC来完成地址空间的切换。为了在切换地址空间的时候不用修改CR3，这里重新映射client’s table page base address (CR3 value)为the HPA of server’s CR3 value in the server’s EPT。VMFUNC触发的 EPT切换会使用后面永远地址翻译的page table改变。基本的思路如下图所示。一般情况下，client和server使用各自的page tables。启动一个新的sever的时候，Subkernel记录下来这个server的server-CR3。Client注册的时候，subkernel向rootkernel请求复制 the client 和 server的两个base EPT，记为EPT-C and EPT-S。Rootkernel重新映射client- CR3到 EPT-S中的HPA of server-CR3，

  ```
  When the client invokes di- rect_server_call interface, the trampoline invokes the VM- FUNC instruction to change the value of EPT pointer from EPT-C to EPT-S. After using EPT-S, client-CR3 will be mapped to the HPA of server-CR3, which means all subse- quent virtual address will be translated by the server page table.
  ```

  也就是说，client调用server的接口的时候就是client在server的虚拟地址空间中执行的。

![](/assets/png/skybridge-va.png)

* Secure Trampoline。SkyBridge使用Trampoline来进行完成一些调用逻辑。因为client的逻辑在server的虚拟地址空间中执行，这样会带来很大的安全隐患，需要必须得是client的代码不能在server的空间中执行，调用运行的智能是server的代码。主要要处理的是client自行构造VMFUNC操作的问题。这里SKyBridge的是动态重写的技术。另外一种安全威胁时调用illegal server，比如调用没有注册的server，illegal client return主要是返回到不是原来的client。为了处理这个问题，SkyBridge准备了一个 calling-key table，在client注册的时候生成。调用的时候带上这个key，返回的时候返回这个key，必须检查这个key是否相同。在Dynamically Rewriting 方面，Paper中有比较多的描述，主要的思路是subkernel会扫描code page，发现VMFUNC的指令，使用functionally-equivalent instructions来替换。

### 0x03 评估

 这里的具体信息可以参看[1].

## XPC: Architectural Support for Secure and Efficient Cross Process Call

### 0x10 核心思路

 XPC的思路是直接使用硬件辅助来实现超高性能的IPC。Paper中总结了IPC主要的几个操作：1. Trap & Restore，机进行调用的操作和调用之后状态恢复的操作，Paper中测试的在seL4上面的测试这两个部分开销在300 cycles左右；2. IPC Logic，执行IPC的逻辑，主要设计到一些权限和资源的检查操作，seL4上面的测试这部分开销在200 cycles左右；3. Process Switch，IPC设计到不同的进程，难免会有一些进程切换的操作，这部分的开销在150-200 cycles；4. Message Transfer，拷贝调用的数据花费的时间(4010 cycles for copying 4KB data)；除了直接的开销之外，由于TLB和Cache等的影响也会带来一些间接的开销。上面的分析总结出要实现高性能的IPC，有两个总结，

````
 two observations: first, a fast IPC that not dependent on the kernel is necessary but still missing. Second, a secure and zero-copying mechanism for passing messages while supporting handover is critical to performance. Our design is based on these two observations.
````

基于上面的观察，XPC的思路是使用硬件辅助来优化性能。

### 0x11 基本设计

 XPC的基本架构如下图。XPC提供了两个基本的原语，x-entry 和 xcall-cap。一个x-entry和一个procedure绑定，即server提供的一个接口，可以被其它的进程调用。一个进程可以创建多个的x-entries。这些x-entries保存在一个x-entry-table中。x-entry-table是内存中的一个区域，另外使用一个寄存器x-entry-table-reg值保存这个区域的基地址，这个思路和x86上面的CR3寄存器保存page table的基地址的思路一致。一个x-entry使用其x-entry ID在x-etrny table中定位。一个接口的调用者使用xcall-cap来调用一个x-entry。xcall-cap即为XPC call capability，表明了每个x-entry的IPC capabilities。提供xcall #reg” 和 “xret”两个指令来进行XPC的调用和返回。

* XPC使用 xcall #reg 调用一个x-entry。XPC engine先通过call-cap bitmap检查caller’s xcall-cap的reading bit，即一个权限的检查；然后engine从x-entry table load这个x-entry，并检查其有效性。之后push一个linkage record到link stack中，这个linkage record用于调用返回，这个link stack为一个线程一个；之后处理器load一个新的page table pointer(在必要的时候需要flush TLB)，并将这个pointer的值设置为 procedure’s entrance address。之后engin在将 caller’s xcall-cap-reg放入到一个寄存器中，用于callee鉴别caller。

* xcall-cap在调用的时候被检查，使用一个bitmap来表示，index i的bit记录一个线程是否有权限调用ID为i的x-entry。这个bitmap记录在一个per-thread一个内存区域中，使用xcall-cap-reg寄存器来记录基地址。xret指令会pop从link stack中pop出linkage record，CPU先检查linkage record的有效bit，然后restore caller的信息。

* Link Stack中，push进去的record记录了calling information (linkage record)。这个也是一个per-thread的内存区域，使用 link-reg寄存器记录基地址，并只能被kernel访问。这个记录中，

  ```
  In our current design, a linkage record includes page table pointer, return address, xcall-cap-reg, seg-list-reg, relay segment and a valid bit. The XPC engine does not save other general registers and leaves to XPC library and applications to handle them. The entries in linkage record can be extended to meet different architectures by obeying a principle that linkage record should maintain information which can not be recovered by user-space.
  ```

* 为了优化性能，这里也使用了cache，称之为XPC Engine Cache。即前面一般操作中需要从内存中获取的信息可以缓存到一个cache中。IPC调用一般也有时间和空间上的局部性。
* 为了减少数据传输带来的开销，XPC引入来relay-seg ( relay segment)。一个relay-seg 就是一段连续的物理内存，replay-seg通过seg-reg来传输其一些信息，这个seg-reg可以有调用者传给被调用者，这样被调用者就可以直接访问数据，而不用进行数据的复制。这里需要kernel来保证relay-seg部分的内存在地址空间中是不会和其它的部分重叠的。

![](/assets/png/xpc-arch.png)

Paper中还讨论了其具体实现的一些信息。

### 0x12 评估

 由于使用来硬件辅助来实现IPC，Paper中这里的性能表现很出色，具体可以参考[2].

## 参考

1. SkyBridge: Fast and Secure Inter-Process Communication for Microkernels, EuroSys ’19.
2. XPC: Architectural Support for Secure and Efficient Cross Process Call, ISCA '19.