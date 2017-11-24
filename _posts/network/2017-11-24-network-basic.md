---
layout: post
title: 网络编程基础
category: 网络
tags: network
keywords: network
description:
---


# 计算机网络

学过计算机网络的人都知道，网络是分层次的。执行流程与邮局非常类似。例如从省寄信到某个乡村，邮局顺序一次是省邮局、市邮局、县邮局、镇邮局。一级一级的，最终送到乡村。iso 给出网络标准是七层，而实际网络则是四层，即物理层、数据链路层、网络层、应用层。传统的网络设备通常工作在数据链路层和网络层，例如交换机、路由器。而目前有些网络控制设备则工作在应用层，这种设备用于对应用层的报文进行分析，进行应用识别和审计。

## TCP and UDP datagram

Question:

In Linux Networking and packets filtering documentation

> If a packet is destined for this box, the packet passes downwards in the diagram, to the INPUT chain. If it passes this, __any processes waiting for that packet will receive it.__

So, suppose there're 3 server apps on a host. Servers A and B are TCP servers, and C is UDP server. Is it true, that if we receive an UDP packet, at IP level this packet is to be delivered for apps A, B, C? Or sockets of apps A & B wouldn't receive this packet at all?

Answer:

> TCP servers and UDP servers operate in very different ways.
>
> At most one TCP server will listen on a given TCP port (corner cases ignored for the sake of simplicity). Connection requests (encapsulated in IP packets) destined for that port are "accepted" by exactly one process (more accurately, accepted by a process that has a file descriptor corresponding to exactly one listening endpoint). The combination of [remote_address,remote_port] and [local_address,local_port] is unique. A TCP server doesn't really receive"packets", it receives a stream of data that doesn't have any specific relationship to the underlying packets that carry the data (packet "boundaries" are not directly visible to the receiving process). And a TCP packet that is neither a connection request nor associated with any existing connection would simply be discarded.
>
> With UDP, each UDP datagram is logically independent and may be received by multiple listening processes. That is, more than one process can bind to the same UDP endpoint and receive datagrams sent to it. Typically, each datagram corresponds to a single IP packet though it is possible for a datagram to be broken into multiple packets for transmission.
>
> So, in your example: no, a server that is listening for TCP requests (a "TCP server") will never receive a UDP packet. The port namespaces for TCP and UDP are completely separate.

## 二层设备

二层设备是工作数据链路层的设备。二层交换机可以识别数据包中的 MAC 地址信息，根据 MAC 地址进行转发，并将这些 MAC 地址与对应的端口记录在自己内部的一个地址表中。具体的工作流程如下：
1. 当交换机从某个端口收到一个数据包，它先读取包头中的源 MAC 地址，这样它就知道源 MAC 地址的机器是连在哪个端口上的；
2. 再去读取包头中的目的 MAC 地址，并在地址表中查找相应的端口；
3. 如表中有与这目的 MAC 地址对应的端口，把数据包直接复制到这端口上；
4. 如表中找不到相应的端口则把数据包广播到所有端口上，当目的机器对源机器回应时，交换机又可以学习一目的 MAC 地址与哪个端口对应，在下次传送数据时就不再需要对所有端口进行广播了。

不断的循环这个过程，对于全网的 MAC 地址信息都可以学习到，二层交换机就是这样建立和维护它自己的地址表。

从二层交换机的工作原理可以推知以下三点：

1. 由于交换机对多数端口的数据进行同时交换，这就要求具有很宽的交换总线带宽，如果二层交换机有 N 个端口，每个端口的带宽是 M，交换机总线带宽超过 N×M，那么这交换机就可以实现线速交换；
2. 学习端口连接的机器的 MAC 地址，写入地址表，地址表的大小（一般两种表示方式：一为 BEFFER RAM，一为 MAC 表项数值），地址表大小影响交换机的接入容量；
3. 还有一个就是二层交换机一般都含有专门用于处理数据包转发的 ASIC （Application specific Integrated Circuit）芯片，因此转发速度可以做到非常快。由于各个厂家采用 ASIC 不同，直接影响产品性能。

以上三点也是评判二三层交换机性能优劣的主要技术参数，这一点请大家在考虑设备选型时注意比较。

## 三层设备与路由技术

三层设备是工作在网络层的设备。路由器是最常用的三层设备，利用不同网络的 ID 号（即 IP 地址）来确定数据转发的地址。IP 地址是在软件中实现的，描述的是设备所在的网络，有时这些第三层的地址也称为协议地址或者网络地址。

路由器工作在 OSI 模型的第三层 --- 网络层操作，其工作模式与二层交换相似，但路由器工作在第三层，这个区别决定了路由和交换在传递包时使用不同的控制信息，实现功能的方式就不同。工作原理是在路由器的内部也有一个表，这个表所标示的是如果要去某一个地方，下一步应该向那里走，如果能从路由表中找到数据包下一步往那里走，把链路层信息加上转发出去；如果不能知道下一步走向那里，则将此包丢弃，然后返回一个信息交给源地址。

路由技术实质上来说不过两种功能：决定最优路由和转发数据包。路由表中写入各种信息，由路由算法计算出到达目的地址的最佳路径，然后由相对简单直接的转发机制发送数据包。接受数据的下一台路由器依照相同的工作方式继续转发，依次类推，直到数据包到达目的路由器。

而路由表的维护，也有两种不同的方式。一种是路由信息的更新，将部分或者全部的路由信息公布出去，路由器通过互相学习路由信息，就掌握了全网的拓扑结构，这一类的路由协议称为距离矢量路由协议；另一种是路由器将自己的链路状态信息进行广播，通过互相学习掌握全网的路由信息，进而计算出最佳的转发路径，这类路由协议称为链路状态路由协议。

由于路由器需要做大量的路径计算工作，一般处理器的工作能力直接决定其性能的优劣。当然这一判断还是对中低端路由器而言，因为高端路由器往往采用分布式处理系统体系设计。

## 第四层交换的原理

OSI 模型的第四层是传输层。传输层负责端对端通信，即在网络源和目标系统之间协调通信。在 IP 协议栈中这是 TCP（一种传输协议）和 UDP（用户数据包协议）所在的协议层。

在第四层中，TCP 和 UDP 标题包含端口号（portnumber），它们可以唯一区分每个数据包包含哪些应用协议（例如 HTTP、FTP 等）。端点系统利用这种信息来区分包中的数据，尤其是端口号使一个接收端计算机系统能够确定它所收到的 IP 包类型，并把它交给合适的高层软件。端口号和设备 IP 地址的组合通常称作 “插口（socket）”。 1 和 255 之间的端口号被保留，他们称为“熟知” 端口，也就是说，在所有主机 TCP/IP 协议栈实现中，这些端口号是相同的。除了 “熟知” 端口外，标准 UNIX 服务分配在 256 到 1024 端口范围，定制的应用一般在 1024 以上分配端口号. 分配端口号的最近清单可以在 RFc1700”Assigned Numbers”上找到。TCP／UDP 端口号提供的附加信息可以为网络交换机所利用，这是第 4 层交换的基础。
