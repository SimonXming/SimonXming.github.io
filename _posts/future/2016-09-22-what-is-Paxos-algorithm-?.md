---
layout: post
title: Paxos算法wiki + 讲解
category: 未来
tags: 分布式 一致性 算法
keywords: 分布式
description:
---

## Wiki

Paxos算法是莱斯利·兰伯特（英语：Leslie Lamport，LaTeX中的“La”）于1990年提出的一种基于消息传递且具有高度容错特性的一致性算法。

### 问题的提出
分布式系统除了能提升整个系统的性能外还有一个重要的特性就是提高系统的可靠性，可靠性指的是当分布式系统中一台或N台机器宕掉后都不会导致系统不可用.

分布式系统是state machine replication的，每个节点都可能是其他节点的快照，这是保证分布式系统高可靠性的关键，而存在多个复制节点就会存在数据不一致的问题，这时一致性就成了分布式系统的核心；

在分布式系统中必须保证：

* 假如在分布式系统中初始是各个节点的数据是一致的，每个节点都顺序执行系列操作，然后每个节点最终的数据还是一致的。

* 一致性算法：用于保证在分布式系统中每个节点都顺序执行相同的操作序列，在每一个指令上执行一致性算法就能够保证最终各个节点的数据都是一致的。

Paxos就是用于解决一致性问题的算法，有多个节点就会存在节点间通信的问题，存在着两种节点通讯模型：共享内存（Shared memory）、消息传递（Messages passing），Paxos是基于消息传递的通讯模型的。 Paxos为2014年图灵奖得主Leslie Lamport在1990年提出的一致性算法，该算法被誉为类似算法中最有效的，Paxos不只适用于分布式系统中，凡是需要达成某种一致性时都可以使用Paxos.

### Paxos概述

作用： Paxos用于解决分布式系统中一致性问题。 在一个Paxos过程只批准一个value，只有被prepare的value且被多数Acceptor接受才能被批准，被批准的value才能被learner.

##### 下面简单描述Paxos的流程：

这样一个场景:

* 有Client一个、Proposer三个、Acceptor三个、Learner一个；

* Client向prepeare提交一个data请求入库，Proposer收到Client请求后生成一个序号1向三个Acceptor（最少两个）发送序号1请求提交议案;

* 假如三个Acceptor收到Proposer申请提交的序号为1的请求，且三个Acceptor都是初次接受到请求，然后向Proposer回复Promise允许提交议案，Proposer收到三个Acceptor（满足过半数原则）的Promise回复后接着向三个Accptor正式提交议案（序号1，value为data）

* 三个Accptor都收到议案（序号1，value为data）请求期间没有收到其他请求，Acceptor接受议案，回复Proposer已接受议案

* 然后向Learner提交议案，Proposer收到回复后回复给Client成功处理请求，Learner收到议案后开始学习议案（存储data）；

Paxos中存在三种角色Proposer（提议者）、Acceptor（决策者）、Learner（议案学习者），整个过程（一个实例或称一个事务或一个Round）分为两个阶段；

##### phase1（准备阶段）
1. Proposer向超过半数（n/2+1）Acceptor发起prepare消息(发送编号)
2. 如果prepare符合协议规则Acceptor回复promise消息，否则拒绝

##### phase2（决议阶段或投票阶段）
1. 如果超过半数Acceptor回复promise，Proposer向Acceptor发送accept消息
2. Acceptor检查accept消息是否符合规则，消息符合则批准accept请求

##### Paxos保证:

1. 只有提出的议案才能被选中，没有议案提出就不会有被选中
2. 多个被提出的议案中只有一个议案会被选中
3. 提案没选中后Learner就可以学习该提案

### 约束条件

__P1: Acceptor必须接受他接收到的第一个提案__

有这约束就会出现一个新问题：当多个议案被多个Proposer同时提出，这时每个Acceptor都接收到了他收到的第一个议案，此时没法选择最终议案。所以就又存在一个新的约束P2；

__P2: 一个提案被选中需要过半数的Acceptor接受。__

假设A为整个Acceptor集合，B为一个超过A一半的Acceptor集合，B为A的子集，C也是一个超过A一半的Acceptor集合，C也是A的子集，有此可知任意两个过半集合中必定有一个共同的成员Acceptor；

此说明了一个Acceptor可以接受不止一个提案，此时需要一个编号来标识每一个提案，提案的格式为：[编号，Value]，编号为不可重复全序的，因为存在着一个一个Paxos过程只能批准一个value这时又推出了一个约束P3；

__P3：当编号K0、Value为V0的提案(即[K0,V0])被过半的Acceptor接受后，今后（同一个Paxos或称一个Round中）所有比K0更高编号且被Acceptor接受的提案，其Value值夜必须为V0。__

因为每个Proposer都可提出多个议案，每个议案最初都有一个不同的Value所以要满足P3就又要推出一个新的约束P4；

__P4：只有Acceptor没有接受过提案Proposer才能采用自己的Value，否者Proposer的Value提案为Acceptor中编号最大的Proposer Value；__

### Paxos流程

这里具体例子来说明Paxos的整个具体流程： 假如有Server1、Server2、Server3这样三台服务器，我们要从中选出leader，这时候Paxos派上用场了。

整个选举的结构图如下：

![image](http://7xnqwp.com1.z0.glb.clouddn.com/Paxos.png)

##### Phase1（准备阶段）

1. 每个Server都向Proposer发消息称自己要成为leader，Server1往Proposer1发、Server2往Proposer2发、Server3往Proposer3发；
2. 现在每个Proposer都接收到了Server1发来的消息但时间不一样，Proposer2先接收到了，然后是Proposer1，接着才是Proposer3；
3. Proposer2首先接收到消息所以他从系统中取得一个编号1，Proposer2向Acceptor2和Acceptor3发送一条，编号为1的消息；接着Proposer1也接收到了Server1发来的消息，取得一个编号2，Proposer1向Acceptor1和Acceptor2发送一条，编号为2的消息； 最后Proposer3也接收到了Server3发来的消息，取得一个编号3，Proposer3向Acceptor2和Acceptor3发送一条，编号为3的消息；
4. 这时Proposer1发送的消息先到达Acceptor1和Acceptor2，这两个都没有接收过请求所以接受了请求返回[2,null]给Proposer1，并承诺不接受编号小于2的请求；
5. 此时Proposer2发送的消息到达Acceptor2和Acceptor3，Acceprot3没有接收过请求返回[1,null]给Proposer2，并承诺不接受编号小于1的请求，但这时Acceptor2已经接受过Proposer1的请求并承诺不接受编号小于的2的请求了，所以Acceptor2拒绝Proposer2的请求；
6. 最后Proposer3发送的消息到达Acceptor2和Acceptor3，Acceptor2接受过提议，但此时编号为3大于Acceptor2的承诺2与Accetpor3的承诺1，所以接受提议返回[3,null];
7. Proposer2没收到过半的回复所以重新取得编号4，并发送给Acceptor2和Acceptor3，然后Acceptor2和Acceptor3都收到消息，此时编号4大于Acceptor2与Accetpor3的承诺3，所以接受提议返回[4,null]；

##### Phase2（决议阶段）

1. Proposer3收到过半（三个Server中两个）的返回，并且返回的Value为null，所以Proposer3提交了[3,server3]的议案；
2. Proposer1收到过半返回，返回的Value为null，所以Proposer1提交了[2,server1]的议案；
3. Proposer2收到过半返回，返回的Value为null，所以Proposer2提交了[4,server2]的议案；
4. Acceptor1、Acceptor2接收到Proposer1的提案[2,server1]请求，Acceptor2承诺编号大于4所以拒绝了通过，Acceptor1通过了请求；
5. Proposer2的提案[4,server2]发送到了Acceptor2、Acceptor3，提案编号为4所以Acceptor2、Acceptor3都通过了提案请求；
6. Acceptor2、Acceptor3接收到Proposer3的提案[3,server3]请求，Acceptor2、Acceptor3承诺编号大于4所以拒绝了提案；
7. 此时过半的Acceptor都接受了Proposer2的提案[4,server2],Larner感知到了提案的通过，Larner学习提案，server2成为Leader；

__一个Paxos过程只会产生一个议案所以至此这个流程结束，选举结果server2为Leader.__

## Explanation

### 数据库高可用性难题

数据库的数据一致和持续可用对电子商务和互联网金融的意义不言而喻，而这些业务在使用数据库时，无论 MySQL 还是 Oracle，都会面临一个艰难的取舍，就是如何处理主备库之间的数据同步。对于传统的主备模式或者一主多备模式，我们都需要考虑的问题，就是与备机保持强同步还是异步复制。

对于强同步模式，要求主机必须把 Redolog 同步到备机之后，才能应答客户端，一旦主备之间出现网络抖动，或者备机宕机，则主机无法继续提供服务，这种模式实现了数据的强一致，但是牺牲了服务的可用性，且由于跨机房同步延迟过大使得跨机房的主备模式也变得不实用。

而对于异步复制模式，主机写本地成功后，就可以立即应答客户端，无需等待备机应答，这样一旦主机宕机无法启动，少量不同步的日志将丢失，这种模式实现了服务持续可用，但是牺牲了数据一致性。这两种方式对应的就是 Oracle 的 Max Protection 和 Max Performance 模式，而 Oracle 另一个最常用的 Max Availability 模式，则是一个折中，在备机无应答时退化为 Max Performance 模式，我认为本质上还是异步复制。

主备模式还有一个无法绕过的问题，就是选主，最简单山寨的办法，搞一个单点，定时 Select 一下主机和各个备机，貌似 MHA 就是这个原理，具体实现细节我就不太清楚了。一个改进的方案是使用类似 ZooKeeper 的多点服务替代单点，各个数据库机器上使用一个 Agent 与单点保持 Lease，主机 Lease 过期后，立即置为只读。改进的方案基本可以保证不会出现双主，而缺点是 ZooKeeper 的可维护性问题，以及多级 Lease 的恢复时长问题（这个本次就不展开讲了，感兴趣的同学请参考这篇文章 Http://oceanbase.org.cn/

### Paxos 协议简单回顾

主备方式处理数据库高可用问题有上述诸多缺陷，要改进这种数据同步方式，我们先来梳理下数据库高可用的几个基本需求：

1. 数据不丢失
2. 服务持续可用
3. 自动的主备切换

使用 Paxos 协议的日志同步可以实现这三个需求，而 Paxos 协议需要依赖一个基本假设，主备之间有多数派机器（N / 2 + 1）存活并且他们之间的网络通信正常，如果不满足这个条件，则无法启动服务，数据也无法写入和读取。

我们先来简单回顾一下 Paxos 协议的内容，首先，Paxos 协议是一个解决分布式系统中，多个节点之间就某个值（提案）达成一致（决议）的通信协议。它能够处理在少数派离线的情况下，剩余的多数派节点仍然能够达成一致。然后，再来看一下协议内容，它是一个两阶段的通信协议，推导过程我就不写了（中文资料请参考这篇 [http://t.cn/R40lGrp](http://t.cn/R40lGrp) ），直接看最终协议内容：


### 第一阶段 Prepare

#### P1a：Proposer 发送 Prepare
Proposer 生成全局唯一且递增的提案 ID（Proposalid，以高位时间戳 + 低位机器 IP 可以保证唯一性和递增性），向 Paxos 集群的所有机器发送 PrepareRequest，这里无需携带提案内容，只携带 Proposalid 即可。

#### P1b：Acceptor 应答 Prepare
Acceptor 收到 PrepareRequest 后，做出 “两个承诺，一个应答”。

两个承诺：
第一，不再应答 Proposalid 小于等于（注意：这里是 <= ）当前请求的 PrepareRequest；
第二，不再应答 Proposalid 小于（注意：这里是 < ）当前请求的 AcceptRequest

一个应答：
返回自己已经 Accept 过的提案中 ProposalID 最大的那个提案的内容，如果没有则返回空值;

注意：这 “两个承诺” 中，蕴含两个要点：

就是应答当前请求前，也要按照 “两个承诺” 检查是否会违背之前处理 PrepareRequest 时做出的承诺；
应答前要在本地持久化当前 Propsalid。

### 第二阶段 Accept

#### P2a：Proposer 发送 Accept
“提案生成规则”：Proposer 收集到多数派应答的 PrepareResponse 后，从中选择 proposalid 最大的提案内容，作为要发起 Accept 的提案，如果这个提案为空值，则可以自己随意决定提案内容。然后携带上当前 Proposalid，向 Paxos 集群的所有机器发送 AccpetRequest。

#### P2b：Acceptor 应答 Accept
Accpetor 收到 AccpetRequest 后，检查不违背自己之前作出的 “两个承诺” 情况下，持久化当前 Proposalid 和提案内容。最后 Proposer 收集到多数派应答的 AcceptResponse 后，形成决议。

这里的 “两个承诺” 很重要，后面也会提及，请大家细细品味。
