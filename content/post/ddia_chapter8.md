---
title: "Designing Data-Intensive Applications 8: The trouble with distributed systems"
date: 2019-01-23T13:05:20+08:00
draft: false
---

# 分布式系统需要思考的问题

这章信息点太碎片化了，其实笔记写了很多，但太过细散化。就利用结尾的summary 精简概括下。

## Summary

分布式系统问题小结：

1. Unliable Network：尝试发包时，包裹肯能丢失或者滞后，同样的，服务器返回的responsne 也可能被丢失或者之后，如果没有收到来自服务端的包裹，是无法确认消息有没有成功传递。e.g. TCP的三次握爪，用于确认客户端，服务端都具备发送接收能力。
2. Unreliable clocks：每个节点的clock可能严重的跟其余的接电脑out of sync。
3. Paused processes：一个进程在任意时间可能暂停一段时间，暂停么有上限，然后被其它节点按照dead处理。比如GC处理。

## 设计一个能tolerate faults的系统

1. detect faults：为了tolerate faults，第一步需要去detect faults，通常办法是依靠timeout 来检查是不是有partial failures 发生。**但timeout 无法区分网络问题还是节点问题。**
2. protocols： nodes依靠网络通信（不可靠）来判断其余node是否alive，需要设计一个protocos 让nodes们来判断。