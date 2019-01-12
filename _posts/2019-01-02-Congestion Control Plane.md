---
layout: page
title: Restructuring Endpoint Congestion Control
tags: [Network]
excerpt_separator: <!--more-->
typora-root-url: ../
---

## Restructuring Endpoint Congestion Control

### 0x00 引言

  这也是一篇比较有意思的Paper，主要讨论的是提出了Congestion Control Plane (CCP)的概念，希望能够实现Wirte-once，run-anywhere，以及可以容易在此的基础之上实现新的拥塞控制算法。CCP的设计中按照下面的设计原则，

* 数据面和拥塞控制隔离；
* 拥塞控制和ACK-clock分离，以一个周期报告ACK的信息，而不是每一个ACK报告一次；
* 可以数据面内实现对每一个ACK的逻辑。

CCP一个基本的架构如下，

![ccp-arch](/assets/images/ccp-arch.png)



### 0x01 使用CCP实现一个算法

  CCP中主要使用的是onCreate和onReport两个回调处理函数。在一个流建立的时候数据面会调用这个onCreate的函数。这个函数会初始化一个数据面的程序。这个数据面的函数会计算出一些关于每个数据包的一些信息，这些信息会汇报给CCP agent。在report的时候，CCP agent调用onReport函数，这个函数根据这个函数计算出新的发送窗口的大小 or 发送速率。CCP的数据面程序使用一个简单的领域特定语言写成，根据每一个ACK来计算一些统计信息(control actions)，一个简单的数据面程序，

```
(def (Report (volatile acked 0) (volatile lost 0))) 
(when true
  (:= Report.acked (+ Report.acked Ack.bytes_acked))
  (:= Report.lost (+ Report.lost Ack.lost_pkts_sample)) (fallthrough))
(when (> Report.lost 0) (report))
```

这个DSL中包含的一些ACK的信息、一些Flow的信息，

![ccp-dsl](/assets/images/ccp-dsl.png)

CCP中的onReport()提供了在用户空间处理来自数据面数据的方法，一个简单的AIMD的算法实现的逻辑在onReport中实现如下，

```python
def onReport(self, report): 
  if report["lost"] > 0:
    self.cwnd = self.cwnd / 2 
  else:
    acked = report["acked"]
    self.cwnd = self.cwnd + acked*MSS/self.cwnd 
    self.update("cwnd", self.cwnd/MSS)
```

把数据面的DSL和CCP的两个Handler的实现组合在一起，下面是一个Slow Start的例子，

```
fn create(...) {
  datapath.install("
   (def (Report (volatile acked 0) (volatile loss 0))) 
   (when true
     (:= Report.acked (+ Report.acked Ack.bytes_acked))) 
     (when (> Micros Flow.rtt_sample_us) (report) (reset))");
}
fn onReport(...) {
  if report.get_field("Report.loss") == 0 {
    let acked = report.get_field("Report.acked"); 
    self.cwnd += acked;
    datapath.update_field(&[("Cwnd", self.cwnd)]);
  } else { 
    /* exit slow start */ 
  } 
}
```

.

### 0x02 实现

  Paper中实现的CCP agent使用Rust语言实现，运行在用户空间之中。一个数据面实现了对CCP的支持之后，它就可以使用在CCP中实现的所有的算法。实现一个支持CCP的数据面要做下面的一些事情，

* 这个数据面应该要实现通过IPC与CCP agent通信的逻辑。数据面将多个的连接的信息通过一个IPC连接汇报给Agent；
* 这个数据应该执行一个用户定义的使用DSL编写成的一个程序，这个程序会在每一次收到ACK或者是超时的时候被调用。

由于要数据面要执行来自其它地方的代码，这里要注意处理就是安全问题和代码中出现的Bug等的问题，为此作者开发了一个libccp库，用于方便开发，

```
To use libccp, the datapath must provide callbacks to functions that: (1) set the window and rate, (2) provide a notion of time, and (3) send an IPC message to CCP. Upon reading a message from CCP, the datapath calls ccp_recv_msg(), which automatically de-multiplexes the message for the correct flow. 
```

.

### 0x03 评估

 这里的详细可以参看[1].

![ccp-perf](/assets/images/ccp-perf.png)

## 参考

1. Restructuring Endpoint Congestion Control, SIGCOMM'18.

