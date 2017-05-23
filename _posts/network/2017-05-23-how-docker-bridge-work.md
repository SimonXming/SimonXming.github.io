---
layout: post
title: Docker 的桥接网络是怎么工作的 及 Docker0 的 “双重身份”
category: 网络
tags: docker0 桥接网络
keywords: Docker
description:
---

Read First: [探索 Docker bridge 的正确姿势，亲测有效！](http://blog.daocloud.io/docker-bridge/)

# Docker 的桥接网络是怎么工作的

我们都知道 docker 支持多种网络，默认网络 bridge 是通过一个网桥进行容器间通信的。

## 主机和容器间的网络连接

进入虚拟机后，在 vagrant 主机上运行ifconfig，就能看到有 4 个网络设备及它们的 IPv4 地址：
* docker0：172.17.0.1
* eth0：10.0.2.15
* eth1：192.168.33.10
* lo：127.0.0.1

其中的 `eth0` 和 `eth1` 是普通的以太网卡，`eth1` 就是我们解除注释的 IP：`192.168.33.10`。
lo是所谓的环回网卡，每台机器都有。它将这台机器/容器绑定到 `127.0.0.1` 的 IP 上，这样子就算没有真实的网卡，也能通过这个 IP 访问自己，对于测试来说尤其方便。最上面的docker0就是我们常说的网桥。怎么知道它是个网桥呢？安装bridge-utils的包就能看到了：

```shell
sudo apt-get install bridge-utils
brctl show
```

网桥设备就好比交换机，可以和其他的网络设备相连接，就像在其他网络设备上拉根网线到这个交换机一样。

进入容器后，运行 ifconfig ，就能够看到有 2 个网络设备及它们的 IPv4 地址：
* eth0：172.17.0.2
* lo：127.0.0.1
它也有自己的 lo ，还有一块以太网卡 eth0 ，目前的 IP 是 172.17.0.2。使用快捷键Ctrl+P然后再Ctrl+Q，就能退出容器并保持它继续运行。在 vagrant 主机上运行route命令，可以看到类似下面这个表格：

Destination|Gateway|Genmask|Flags|Metric|Ref|Use|Iface
-----------|-------|-------|-----|------|---|---|-----
default|10.0.2.2|0.0.0.0|UG|0|0|0|eth0
10.0.2.0|*|255.255.255.0|U|0|0|0|eth0
172.17.0.0|*|255.255.0.0|U|0|0|0|docker0
192.168.33.0|*|255.255.255.0|U|0|0|0|eth1

这个就是 vagrant 主机的路由表。我们重点看一下 `172.17.0.0` 这一行。它的 Genmask 为 `255.255.0.0`，就意味着 `255.255` 对应着的 IP `172.17` 是网络地址，而 Genmask 中 `0.0` 对应着的 IP `0.0` 是主机地址。整行的意思就是当目标地址是 `172.17.*.*` 的时候，匹配这条路由规则。还有一种写法是172.17.0.0/16。当 Gateway 不为 `*` 号时，那就会路由到 Gateway 去，否则就路由到 Iface 去。刚才我们知道新容器的 IP 是 `172.17.0.2`，所以当 vagrant 主机上的某个数据包的地址是这个新容器的 IP 时，就会匹配这条路由规则，由 docker0 来接受这个数据包。如果数据包的地址都不匹配这些规则，就送到default那一行的Gateway里。

那么 docker0 在接收数据包之后，又会送到哪里去呢？我们在 vagrant 主机再次运行brctl show，便能看到 docker0 这个网桥有所变化。它的 interfaces 里增加了一个vethXXX，在我的机器上叫 vethd6d3942。在 vagrant 主机再次运行ifconfig，我们也能看到这一块新增的 VETH 虚拟网卡。实际上每启动一个容器，docker 便会增加一个叫vethXXX的设备，并把它连接到 docker0 上，于是 docker0 就可以把收到的数据包发给这个 VETH 设备。VETH 设备总是成对出现，一端进去的请求总会从 peer 也就是另一端出来，这样就能将一个 namespace 的数据发往另一个 namespace，就像虫洞一样。那么现在这一端是vethd6d3942，它的另一端是哪儿呢？运行这个命令（记得把 VETH 设备名改成你自己主机上的设备名）：

```shell
ethtool -S vethd6d3942
```

可以看到这个 VETH 设备的peer_ifindex是某个数字，在我的机器上是5。这个 ifindex 是一个网络接口的唯一识别编号。通过docker exec -it ubuntu bash进入容器里，然后运行：

```shell
ip link
```

可以看到5: eth0，原来跨越 namespace 跑到容器里头来啦。这就是主机上的 VETH 设备能跟容器内部通信的原因。每当新启动一个容器，主机就会增加一对 VETH 设备，把一个连接到 docker0 上，另一个挂载到容器内部的 eth0 里。

## IP 和 mac 地址映射
还有一个问题：每个网络设备都有自己的 mac 地址，通过 ip 怎么能找到它呢？在容器外运行这个命令：

```shell
arp -n
```
我们就能看到Address和HWaddress，它们分别对应着 IP 地址和 mac 地址，这样就匹配起来了。到容器里ifconfig一下，看看172.17.0.2的 mac 地址，是不是和主机arp -n运行结果中172.17.0.2那行的 mac 地址一样呢？

## Docker0 的 “双重身份”

如何理解 Docker0。下图中我们给出了 Docker0 的双重身份，并对比物理交换机，我们来理解一下 Docker0 这个软网桥。

![docker0](http://tonybai.com/wp-content/uploads/docker-single-host-networking-docker0.jpg)

1. 从容器视角，网桥（交换机）身份

docker0 对于通过 veth pair “插在” 网桥上的 container1 和 container2 来说，首先就是（第一个身份）一个二层的交换机的角色：泛洪、维护 cam 表，在二层转发数据包；同时由于 docker0 自身也具有 mac 地址（这个与纯二层交换机不同），并且绑定了 ip(这里是 172.17.0.1)，因此在 container 中还作为 container default 路由的默认 Gateway 而存在。

2. 从宿主机视角，网卡身份

物理交换机提供了由硬件实现的高效的背板通道，供连接在交换机上的主机高效实现二层通信；对于开启了三层协议的物理交换机而言，其 ip 路由的处理 也是由物理交换机管理程序提供的。

对于 docker0 而言，其负责处理二层交换机逻辑以及三层的处理程序其实就是宿主机上的 Linux 内核 tcp/ip 协议栈程序。

而从宿主机来看，所有 docker0 从 veth（只是个二层的存在，没有绑定 ipv4 地址）接收到的数据包都会被宿主机看成从 docker0 这块网卡（第二个身份，绑定 172.17.0.1) 接收进来的数据包，尤其是在进入三层时，宿主机上的 iptables 就会 对 docker0 进来的数据包按照 rules 进行相应处理（通过一些内核网络设置也可以忽略 docker0 brigde 数据的处理）。

在后续的 Docker 容器网络通信流程分析中，docker0 将在这两种身份间来回切换。