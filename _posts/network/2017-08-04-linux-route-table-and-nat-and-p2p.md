---
layout: post
title: route table 和 NAT 和 p2p
category: 网络
tags: route iptable NAT
keywords: 网络
description:
---

# route table

## 路由表简介

路由表（routing table）是一个存储在路由器或者联网计算机中的电子表格或类数据库。路由表存储着指向特定网络地址的路径（在有些情况下，还记录有路径的路由度量值）。路由表中含有网络周边的拓扑信息。路由表建立的主要目标是为了实现路由协议和静态路由选择。

在Linux系统中，设置路由通常是为了解决以下问题：该Linux系统在一个局域网中，局域网中有一个网关，能够让机器访问Internet，那么就需要将这台机器的IP地址设置为Linux机器的默认路由。要注意的是，直接在命令行下执行route命令来添加路由，不会永久保存，当网卡重启或者机器重启之后，该路由就失效了;可以在/etc/rc.local中添加route命令来保证该路由设置永久有效。

使用实例
```shell
# Linux
route # 或 route -n
# MacOS
netstat -nr

# route 
Destination     Gateway         Genmask         Flags Metric Ref     Use Iface  
192.168.0.0     *               255.255.255.0    U     0      0        0  eth0  
169.254.0.0     *               255.255.0.0      U     0      0        0  eth0  
default         192.168.0.1     0.0.0.0          UG    0      0        0  eth0 
```

route 命令的输出项说明：

* Gateway 网关地址，”*” 表示目标是本主机所属的网络，不需要路由
* Genmask 网络掩码
* Flags 标记。一些可能的标记如下：

    * U Up表示此路由当前为启动状态
    * H Host，表示此网关为一主机
    * G Gateway，表示此网关为一路由器
    * R Reinstate Route，使用动态路由重新初始化的路由
    * D Dynamically,此路由是动态性地写入
    * M Modified，此路由是由路由守护程序或导向器动态修改
    * ! 表示此路由当前为关闭状态

* Metric 路由距离，到达指定网络所需的中转数（linux 内核中没有使用）
* Ref 路由项引用次数（linux 内核中没有使用）
* Use 此路由项被路由软件查找的次数
* Iface 该路由表项对应的输出接口
> 备注：
> route -n (-n 表示不解析名字,列出速度会比route 快)

## 3 种路由类型

### 主机路由

主机路由是路由选择表中指向单个IP地址或主机名的路由记录。主机路由的Flags字段为H。例如，在下面的示例中，本地主机通过IP地址192.168.1.1的路由器到达IP地址为10.0.0.10的主机。

```shell
Destination    Gateway       Genmask Flags     Metric    Ref    Use    Iface
-----------    -------     -------            -----     ------    ---    ---    -----
10.0.0.10     192.168.1.1    255.255.255.255   UH       0    0      0    eth0
```

### 网络路由

网络路由是代表主机可以到达的网络。网络路由的Flags字段为N。例如，在下面的示例中，本地主机将发送到网络192.19.12.0的数据包转发到IP地址为192.168.1.1的路由器。

```shell
Destination    Gateway       Genmask          Flags    Metric    Ref     Use    Iface
-----------    -------        -------         -----    -----   ---    ---    -----
192.19.12.0     192.168.1.1    255.255.255.0      UN      0       0     0    eth0
```

它的 Genmask 为 255.255.255.0，就意味着 255.255.255 对应着的 IP 192.19.12 是网络地址，而 Genmask 中 .0 对应着的 IP .0 是主机地址。整行的意思就是当目标地址是 192.19.12.* 的时候，匹配这条路由规则。还有一种写法是192.19.12.0/24。当 Gateway 不为 * 号时，那就会路由到 Gateway 去，否则就路由到 Iface 去。

### 默认路由

当主机不能在路由表中查找到目标主机的IP地址或网络路由时，数据包就被发送到默认路由（默认网关）上。默认路由的Flags字段为G。例如，在下面的示例中，默认路由是IP地址为192.168.1.1的路由器

```shell
Destination    Gateway       Genmask    Flags     Metric    Ref    Use    Iface
-----------    -------     -------      -----     ------    ---    ---    -----
default       192.168.1.1     0.0.0.0    UG       0         0      0    eth0
```

# 关于路由过程

## 详情

下面以一个简单的PING示例解释一下路由的整个过程。
有两个原则需要记住：
* 二层网络是通过MAC地址进行交换
* 三层路由通过IP选路。

![路由过程](/assets/img/post/20170804/路由过程.png)

一个ICMP的以太网格式报文：

`|目的MAC|源MAC|目的IP|源IP|ICMP包|`

主机A发送一个ICMP包到主机B, 此时目的IP为10.10.0.2，源IP为192.168.0.2。 主机A发现，这个IP和我不在一个网段，那就需要进行IP选路了。此时它就只能把包发往直接的网关了(192.168.0.1) ，主机A和网关在同一个网段，通过MAC地址进行交换，所以目的MAC就是MACRA， 源Mac就是MACHOSTA。
因此从主机A发出的一个以太网报文格式为：

`|MACRA|MACHOSTA|10.10.0.2|192.168.0.2|ICMP包`

路由器在接口A处收到以太网包进行分析，选出目的IP，根据自身的路由表发现，这个数据包要经过接口B发出啊。于是就在路由器接口B封装数据包

`|MACHOSTB|MACRB|10.10.0.2|192.168.0.2|ICMP包`

主机B在二层收到从路由器接口B发送的以太网报文进行解析发现目的IP是自己的，然后开始生成回复。首先发现目的IP是192.168.0.2.这货和我不是一个网段的啊，那我就把数据包先发给网关，让网关帮我邮递。于是封装出下面的以太网包：

`|MACRB|MACHOSTB|192.168.0.2|10.10.0.2|ICMP`

路由器从接口B收到数据包之后，同样进行分析之后知道自己需要通过接口A发出。于是封装的以太网包：

`|MACHOSA|MACRA|192.168.0.2|10.10.0.2|ICMP`

终于回复的数据包被主机A收到，整个ICMP过程完成。

有一个非常重要的原则要记住：
* 在数据跳转的过程中，改变的是MAC，IP是始终是没有改变的。

# 关于 NAT

![NAT 路由过程](/assets/img/post/20170804/完整路由过程.png)

上图是我们家里常用的路由器连接示例，HOSTA和HOSTB连接到路由器的LAN口，当主机A访问8.8.8.8时，根据上面讲的路由过程，我们知道到达路由器的以太网包为：

`|MACRA|HOSTA|8.8.8.8|192.168.1.2|ICMP包`

此时路由器通过WAN口将报文发送到公网上去，按这常理它的数据包格式应该是：

`|下一跳路由MAC|路由器A的WAN口的MAC|8.8.8.8|192.168.1.2|ICMP包`

我们在深入思考一下，假设这样的一个数据包发送到公网上之后，确定还能按原路找回来吗？ 大家的路由器网关都是192.168.1.1 ，数据如果回来之后根本就不知道发送到哪一个路由器了。

这个时候就出现了NAT的技术，如果想让回复的数据包能够返回，那么我就需要更改源IP地址啊，于是路由器在从WAN口路由出去之后更改源IP为WAN口的IP，同时记录NAT转换的信息，保证回复的数据能够投递到指定的目标。这种技术叫做SNAT。同理还有一个叫做DNAT，也就是更改目的地址IP。 后边会在介绍k8s路由过程中解释。 只需要记住SNAT是路由之后进行的，DNAT是路由之前进行的即可。

## Iptables原理

Iptables 是 Linux 内核模块 netfilter 的应用程序，通过 iptables 这个应用程序来控制 netfilter 内核的行为。Linux的iptables默认是4表5链，当然也可以自定义自己的规则链。

* 4表： nat， manage， raw， filter， 
* 5链： prerouting， input， output， forward, postrouting 

一般情况下我们只需要关心nat和filter这个两个表。Nat表可以控制非本地的网络包，filter可以控制路由到本地的数据包。

![NAT 路由过程](/assets/img/post/20170804/iptable.png)


### iptables 传输数据包的过程

1. 当一个数据包进入网卡时，它首先进入 PREROUTING 链，内核根据数据包目的 IP 判断是否需要转送出去。 
2. 如果数据包就是进入本机的，它就会沿着图向下移动，到达 INPUT 链。数据包到了 INPUT 链后，任何进程都会收到它。本机上运行的程序可以发送数据包，这些数据包会经过 OUTPUT 链，然后到达 POSTROUTING 链输出。 
3. 如果数据包是要转发出去的，且内核允许转发，数据包就会如图所示向右移动，经过 FORWARD 链，然后到达 POSTROUTING 链输出。

### iptables 的规则表和链：

* 表（tables）提供特定的功能，iptables 内置了 4 个表，即 filter 表、nat 表、mangle 表和 raw 表，分别用于实现包过滤，网络地址转换、包重构 (修改) 和数据跟踪处理。
* 链（chains）是数据包传播的路径，每一条链其实就是众多规则中的一个检查清单，每一条链中可以有一条或数条规则。当一个数据包到达一个链时，iptables 就会从链中第一条规则开始检查，看该数据包是否满足规则所定义的条件。如果满足，系统就会根据该条规则所定义的方法处理该数据包；否则 iptables 将继续检查下一条规则，如果该数据包不符合链中任一条规则，iptables 就会根据该链预先定义的默认策略来处理数据包。

### 规则表：

1. filter 表——三个链：INPUT、FORWARD、OUTPUT
作用：过滤数据包  内核模块：iptables_filter.
2. Nat 表——三个链：PREROUTING、POSTROUTING、OUTPUT
作用：用于网络地址转换（IP、端口） 内核模块：iptable_nat
3. Mangle 表——五个链：PREROUTING、POSTROUTING、INPUT、OUTPUT、FORWARD
作用：修改数据包的服务类型、TTL、并且可以配置路由实现 QOS 内核模块：iptable_mangle(别看这个表这么麻烦，咱们设置策略时几乎都不会用到它)
4. Raw 表——两个链：OUTPUT、PREROUTING
作用：决定数据包是否被状态跟踪机制处理  内核模块：iptable_raw

### 规则链：
1. INPUT——进来的数据包应用此规则链中的策略
2. OUTPUT——外出的数据包应用此规则链中的策略
3. FORWARD——转发数据包时应用此规则链中的策略
4. PREROUTING——对数据包作路由选择前应用此链中的规则
（记住！所有的数据包进来的时侯都先由这个链处理）
5. POSTROUTING——对数据包作路由选择后应用此链中的规则
（所有的数据包出来的时侯都先由这个链处理）

* [扩展阅读: iptable 详情](http://www.cnblogs.com/metoy/p/4320813.html)

# 什么是 p2p？

## 简介

p2p（peer to peer）可以定义成终端之间通过直接交换来共享计算机资源和服务，而无需经过服务器的中转。它的好处是显而易见的，不用服务器中转，不需要受限于服务器的带宽，而且大大减轻了服务器的压力。p2p 的应用包括 IM（qq，MSN），bittorrent 等等。

## 为什么要 NAT 穿越

要实现 p2p，我们要克服的就是 NAT 穿越。在现在的互联网环境下，一个终端一般都在一个 NAT 内，NAT 会有一个网关路由，对外暴露一个 public IP，那么两个都在 NAT 的终端怎么通信呢？我们不知道对方的内网 IP，即使把消息发到对方的网关，然后呢？网关怎么知道这条消息给谁，而且谁允许网关这么做了？

最常见的 NAT 穿透是基于 UDP 的技术，如 RFC3489 中定义的 STUN 协议。
STUN，首先在 RFC3489 中定义，作为一个完整的 NAT 穿透解决方案，英文全称是 Simple Traversal of UDP Through NATs，即简单的用 UDP 穿透 NAT。

在新的 RFC5389 修订中把 STUN 协议定位于为穿透 NAT 提供工具，而不是一个完整的解决方案，英文全称是 Session Traversal Utilities for NAT，即 NAT 会话穿透效用。RFC5389 与 RFC3489 除了名称变化外，最大的区别是支持 TCP 穿透。
TURN，首先在 RFC5766 中定义，英文全称是 Traversal Using Relays around NAT:Relay Extensions to Session Traversal Utilities for NAT，即使用中继穿透 NAT:STUN 的扩展。简单的说，TURN 与 STURN 的共同点都是通过修改应用层中的私网地址达到 NAT 穿透的效果，异同点是 TURN 是通过两方通讯的 “中间人” 方式实现穿透。

## 一个容易的问题

NAT 内的设备怎么和公网服务器通信？

假设路由器 ip 为 1.2.3.4，公网服务器 ip 为 5.6.7.8，内网机器192.168.0.240:5060首先发给路由器1.2.3.4，路由器分配一个端口，比如说 54333，然后路由器代替内网机器发给服务器，即 1.2.3.4:54333 -> 5.6.7.8:80，此时 路由器会在映射表上留下一个 “洞”，来自5.6.7.8:80发送到 1.2.3.4的 54333 端口的包都会转发到 192.168.0.250:5060

但不是所有发往 1.2.3.4:54333 的包都会被转发过去，不同的 NAT 类型有不同的做法。下面我们来看几种 NAT 的类型：

### NAT 类型之一：Full Cone

全锥形 NAT

IP、端口都不受限。只要客户端由内到外打通一个洞之后（NatIP:NatPort –> A:P1），其他 IP 的主机 (B) 或端口 (A:P2) 都可以使用这个洞发送数据到客户端。

### NAT 类型之二：Restricted Cone

受限锥形 NAT

IP 受限，端口不受限。当客户端由内到外打通一个洞之后 (NatIP:NatPort –> A:P1)，A 机器可以使用他的其他端口（P2）主动连接客户端，但 B 机器则不被允许。

### NAT 类型之三：Restricted Port Cone

端口受限锥型

IP、端口都受限。返回的数据只接受曾经打洞成功的对象（A:P1），由 A:P2、B:P1 发起的数据将不被 NatIP:NatPort 接收。

### NAT 类型之四：Symmetric NAT

对称型 NAT

对称型 NAT 具有端口受限锥型的受限特性。但更重要的是，他对每个外部主机或端口的会话都会映射为不同的端口（洞）。只有来自相同的内部地址（IP:PORT）并且发送到相同外部地址（X:x）的请求，在 NAT 上才映射为相同的外网端口，即相同的映射。

举个例子：

* client 访问 A:p1 是这样的路径：Client --> NatIP:Pa1 --> A:P1
* client 访问 A:p2 是这样的路径：Client --> NatIP:Pa2 --> A:P2
* (而在前面的三种 NAT 中，只要 client 不变，那么留在路由器上的 “洞” 就不会变，symmetric NAT 会变，端口变)

## 怎么确定自己的 NAT 类型

为什么要知道自己的 NAT 类型？这为之后的打洞做准备。RFC 专门定义了一套协议来做这件事（RFC 5389），这个协议的名字叫 STUN（Session Traversal Utilities for NAT），它的算法输出是:

1. Public ip and port
2. 防火墙是否设置
3. 是否在 NAT 之后以及 NAT 的类型

![STUN](http://zyearnpic.qiniudn.com/STUN.png)

```shell
$ pip install pystun
$ pystun
NAT Type: Symmetric NAT
External IP: 58.246.6.66
External Port: 7427
```

## 穿透(打洞)策略

问题：有两个需要互联的client A和client B

方案：

* A、B分别与stun server交互获得自己的NAT类型
* A、B连接一个公网服务器（turn server，RFC5766），把自己的NAT发给turn server，此时turn server发现A和B想要互连，把对方的ip，port，NAT类型发给对方
* Client根据自身NAT类型做出相应的策略。
![stun](http://zyearnpic.qiniudn.com/turn.png)

### If {有一方对称NAT}

因为每一次连接端口都不一样，所以对方无法知道在对称NAT的client下次用什么端口。 无法完全实现p2p传输（预测端口除外），需要turn server做一个relay，即所有消息通过turn server转发

### Else if {有一方是Full Cone}

一方通过与full cone的一方的public ip和port直接与full cone通信，实现了p2p通信。

### Else if {有一方是受限型NAT(两种)}

受限型一方向对方发送“打洞包”，比如”punching…”，对方收到后回复一个指定的命令，比如”end punching”，通信开始。这样做理由：受限型一方需要让IPA:portA的包进入，需要先向IPA：portA发包。实现了p2p通信。

## 对称NAT实现p2p的一种方法

我们通过上述的讨论可知，symmetric NAT好像不能实现p2p啊？其实不然，能实现，但代价太高，这个方法叫端口预测。

基本思路：

1. A向B的所有port(0~65535)发包，让路由器知道来自B的所有端口都转发到A
2. B向A的所有port(0~65535)发包，让路由器知道来自A的所有端口都转发到B
3. 于是连接完成了