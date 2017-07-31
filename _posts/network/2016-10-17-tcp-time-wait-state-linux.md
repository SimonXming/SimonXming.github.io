---
layout: post
title: 不要开启 net.ipv4.tcp_tw_recycle 选项
category: 网络
tags: Linux 网络 优化
keywords: Linux
description:
---

### 引言

> If you see a lot of error: too many open files in your log, you should optimize your system. This tutorial applies to all shadowsocks servers (Python, libev, etc).

> On Debian 7:
> Create /etc/sysctl.d/local.conf with the following content:

```shell
...
# reuse timewait sockets when safe
net.ipv4.tcp_tw_reuse = 1
# turn off fast timewait sockets recycling
net.ipv4.tcp_tw_recycle = 0
...

# Then
sysctl --system
```
翻译来源: [tcp-time-wait-state-linux](https://vincent.bernat.im/en/blog/2014-tcp-time-wait-state-linux)

### What is net.ipv4.tcp_tw_recycle?

Linux kernel 文档中关于 `net.ipv4.tcp_tw_recycle` 的描述并不能清楚的告诉我们它做了什么。
> Enable fast recycling TIME-WAIT sockets. Default value is 0. It should not be changed without advice/request of technical experts.

和它相关的另一个配置项`net.ipv4.tcp_tw_reuse`的描述稍微丰富了一点，但基本上还是一样的。
> Allow to reuse TIME-WAIT sockets for new connections when it is safe from protocol viewpoint. Default value is 0. It should not be changed without advice/request of technical experts.

这样缺乏官方文档的结果，就是有许多调优指南建议为了将这两个配置项均设置为 1 ，他们说这样可以减少处于 TIME-WAIT 状态下连接的数量。然而，就像tcp(7)手册上说明的，`net.ipv4.tcp_tw_recycle`选项因为无法处理来自在同一个 NAT 设备后的两个不同计算机的连接，因此对面向生产环境的服务器是有问题的，这个问题很难察觉到的潜在威胁。

如同手册的注脚上说的，尽管 ipv4 在它的名字里，`net.ipv4.tcp_tw_recycle` 也会影响到 IPv6。同时，记住我们正在查看 Linux 系统的 TCP stack。这和Netfilter connection tracking完全没关系，Netfilter connection tracking可能会调整为其他方式。

### About TIME-WAIT state:

我们先后退一小步，然后仔细看一下 TIME-WAIT state, 它是什么？查看下面这个 TCP state 图像。

![tcp-state-steps](http://img.blog.csdn.net/20141015155713390)

![tcp-state-diagram](http://7xnqwp.com1.z0.glb.clouddn.com/tcp-state.jpg)

只有主动(首先)关闭连接的一方才会到达 TIME-WAIT state. 接收关闭请求的一方通常会许可(permits)快速清除掉链接。

你可用通过 `ss -tan` 查看当前连接状态

```shell
$ ss -tan | head -5
LISTEN     0  511             *:80              *:*     
SYN-RECV   0  0     192.0.2.145:80    203.0.113.5:35449
SYN-RECV   0  0     192.0.2.145:80   203.0.113.27:53599
ESTAB      0  0     192.0.2.145:80   203.0.113.27:33605
TIME-WAIT  0  0     192.0.2.145:80   203.0.113.47:50685
```

### Purpose

There are two purposes for the TIME-WAIT state:

1. 协议在关闭连接的四次握手过程中，最终的 ACK 是由主动关闭连接的一端（后面统称 A 端）发出的，如果这个 ACK 丢失，对方（后面统称 B 端）将重发出最终的 FIN，因此 A 端必须维护状态信息（TIME_WAIT）允许它重发最终的 ACK。如果 A 端不维持 TIME_WAIT 状态，而是处于 CLOSED 状态，那么 A 端将响应 RST 分节，B 端收到后将此分节解释成一个错误（在 java 中会抛出 connection reset 的 SocketException)。
因而，要实现 TCP 全双工连接的正常终止，必须处理终止过程中四个分节任何一个分节的丢失情况，主动关闭连接的 A 端必须维持 TIME_WAIT 状态 。
![If the remote end stays in LAST-ACK state because the last ACK was lost, opening a new connection with the same quadruplet will not work.](http://7xnqwp.com1.z0.glb.clouddn.com/connect-close-proper.png)

2. 允许老的重复分节在网络中消逝
TCP 分节可能由于路由器异常而 “迷途”，在迷途期间，TCP 发送端可能因确认超时而重发这个分节，迷途的分节在路由器修复后也会被送到最终目的地，这个迟到的迷途分节到达时可能会引起问题。在关闭 “前一个连接” 之后，马上又重新建立起一个相同的 IP 和端口之间的 “新连接”，“前一个连接” 的迷途重复分组在 “前一个连接” 终止后到达，而被 “新连接” 收到了。为了避免这个情况，TCP 协议不允许处于 TIME_WAIT 状态的连接启动一个新的可用连接，因为 TIME_WAIT 状态持续 2MSL，就可以保证当成功建立一个新 TCP 连接的时候，来自旧连接重复分组已经在网络中消逝。

![Due to a shortened TIME-WAIT state, a delayed TCP segment has been accepted in an unrelated connection.](http://7xnqwp.com1.z0.glb.clouddn.com/duplicate-segment.png)

RFC 793 需要 TIME-WAIT state 至少持续 2*MSL 的事件. 在 Linux 系统中, 这段延时事件是不可变的而且定义在了 include/net/tcp.h 文件中为1分钟e:
```c
#define TCP_TIMEWAIT_LEN (60*HZ) /* how long to wait to destroy TIME-WAIT
                                  * state, about 60 seconds     */
```
已经有人 [把时间变成可变的值](http://comments.gmane.org/gmane.linux.network/244411) 但是被拒绝了。

### Problems
现在，我们发现这个状态在服务器处理很多连接时可能是很烦人的。这里有三个方面的问题：
* 处于 TIME-WAIT 状态下的连接会阻止产生相同种类的新连接。(the slot taken in the connection table preventing new connections of the same kind)
* 在内核中由于 socket structure 产生的内存占用
* 以及，额外的 CPU 消耗

The result of ss -tan state time-wait | wc -l is not a problem per se!

### Connection table slot

连接处于 TIME-WAIT 状态时可以在 connection table 中存在一分钟。这意味着，另一个有相同套接字 (quadruplet) (source address, source port, destination address, destination port) 的连接是无法存在的。

对于一个 web 服务器，连接目的地地址和端口很大可能是不会变化的。如果你的 web server 在了一个 L7 负载均衡器后面，那么它的来源地址也将会是不变的。在 Linux 中，客户端程序端口一般会在30000个端口中间分配(this can be changed by tuning net.ipv4.ip_local_port_range)。这也就意味着，每分钟在web 服务器和负载均衡器之间只能建立30000个连接，也就是500个连接每秒。

如果 TIME-WAIT sockets 在客户端一侧的话，这样的问题是很容易检测到的。客户端调用 connect() 方法将会返回 EADDRNOTAVAIL 并且应用程序会打印一些错误日志。在服务器端，因为没有日志和 counter 的帮助，问题就复杂的多。很难说你可以搞出一些东西查出套接字的使用情况。

```shell
$ ss -tan 'sport = :80' | awk '{print $(NF)" "$(NF-1)}' | sed 's/:[^ ]*//g' | sort | uniq -c
    696 10.24.2.30 10.33.1.64
   1881 10.24.2.30 10.33.1.65
   5314 10.24.2.30 10.33.1.66
   5293 10.24.2.30 10.33.1.67
   3387 10.24.2.30 10.33.1.68
   2663 10.24.2.30 10.33.1.69
   1129 10.24.2.30 10.33.1.70
  10536 10.24.2.30 10.33.1.73
```

The solution can be done in several ways (in the order of difficulty to setup):
* use more client ports by setting `net.ipv4.ip_local_port_range` to a wider range,
* use more server ports by asking the web server to listen to several additional ports (81, 82, 83, …),
* use more client IP by configuring additional IP on the load balancer and use them in a round-robin fashion,
* use more server IP by configuring additional IP on the web server.

当然，最后的解决方案是调整 `net.ipv4.tcp_tw_reuse` and `net.ipv4.tcp_tw_recycle`. 现在先别这样做，我们接下来会讨论如何调整这些配置。


### Memory

With many connections to handle, leaving a socket open for one additional minute may cost your server some memory. For example, if you want to handle about 10,000 new connections per second, you will have about 600,000 sockets in the TIME-WAIT state. How much memory does it represent? Not that much!

首先从应用的角度来看, a TIME-WAIT socket does not consume(占用) any memory: the socket has been closed. 
首先从系统内核的角度来看, a TIME-WAIT socket is present in three structures (for three different purposes):

1. A hash table of connections, named the “TCP established hash table” (尽管包含在其中的连接有其他状态的) is used to locate an existing connection, for example when receiving a new segment.

Each bucket of this hash table contains both a list of connections in the `TIME-WAIT` state and a list of regular active connections. The size of the hash table depends on the system memory and is printed at boot:

```shell
$ dmesg | grep "TCP established hash table"
[    0.169348] TCP established hash table entries: 65536 (order: 8, 1048576 bytes)
```

It is possible to override it by specifying the number of entries on the kernel command line with the `thash_entries` parameter.

在这个链接列表里的每个处于 `TIME-WAIT` 状态的连接由 `tcp_timewait_sock` 结构表示, 而其他状态的连接由 `tcp_sock` 结构表示:

<script src="https://gist.github.com/SimonXming/555c8087948f2fbb6acbb6c71fe7b691.js"></script>

2. A set of lists of connections, called the “death row”, is used to expire the connections in the `TIME-WAIT` state. They are ordered by how much time left before expiration.

It uses the same memory space as for the entries in the hash table of connections. This is the struct `hlist_node tw_death_node` member of struct `inet_timewait_sock8`.

3. A hash table of bound ports, holding the locally bound ports and the associated parameters, is used to determine if it is safe to listen to a given port or to find a free port in the case of dynamic bind. The size of this hash table is the same as the size of the hash table of connections:

```shell
$ dmesg | grep "TCP bind hash table"
[    0.169962] TCP bind hash table entries: 65536 (order: 8, 1048576 bytes)
```
hash 表中的每个元素都是一个结构体 `inet_bind_socket`，对每个本地绑定的端口都有一个对应的元素。一个 `TIME-WAIT` 状态的对于 web 服务器的连接本地绑定的端口是 80 并且与其同级的 `TIME-WAIT` 状态连接共享相同的 entry。另外，到远程服务器的连接在本地绑定同样的随机端口并且不会共享它的 entry。


So, we are only concerned by the space occupied by struct `tcp_timewait_sock` and struct `inet_bind_socket`. There is one struct `tcp_timewait_sock` for each connection in the `TIME-WAIT` state, inbound or outbound. There is one dedicated struct `inet_bind_socket` for each outbound connection and none for an inbound connection.

A struct `tcp_timewait_sock` is only 168 bytes while a struct `inet_bind_socket` is 48 bytes:

```shell
$ sudo apt-get install linux-image-$(uname -r)-dbg
[...]
$ gdb /usr/lib/debug/boot/vmlinux-$(uname -r)
(gdb) print sizeof(struct tcp_timewait_sock)
 $1 = 168
(gdb) print sizeof(struct tcp_sock)
 $2 = 1776
(gdb) print sizeof(struct inet_bind_bucket)
 $3 = 48
```

So, if you have about 40,000 inbound connections in the `TIME-WAIT` state, it should eat less than 10MB of memory. If you have about 40,000 outbound connections in the `TIME-WAIT` state, you need to account for 2.5MB of additional memory. Let’s check that by looking at the output of slabtop. Here is the result on a server with about 50,000 connections in the `TIME-WAIT` state, 45,000 of which are outbound connections:

```shell
sudo slabtop -o | grep -E '(^  OBJS|tw_sock_TCP|tcp_bind_bucket)'
  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME                   
 50955  49725  97%    0.25K   3397       15     13588K tw_sock_TCP            
 44840  36556  81%    0.06K    760       59      3040K tcp_bind_bucket
```

这不需要改变任何事情：`TIME-WAIT` 连接使用的内存非常小。如果你的服务器每秒需要处理几千的新请求，则需要更多的内存才能有效地将数据推送到客户端。`TIME-WAIT` 连接带来的开销是可以忽略的。

### CPU

从 cpu 角度来看，寻找空闲的本地端口有可能是有些昂贵的操作。这个工作由 `inet_csk_get_port()` 函数完成，它使用一个锁并且在本地绑定端口上重复查询知道一个空闲端口被找到。在 hash table 有大量的记录在你有大量 outbound connections 时通常不是个问题(like ephemeral connections to a memcached server)：这些连接通常会共享同样的数据，`inet_csk_get_port()` 函数可以快速的顺序的查询到空闲端口。

### Other solutions

If you still think you have a problem with `TIME-WAIT` connections after reading the previous section, there are three additional solutions to solve them:
* disable socket lingering,
* `net.ipv4.tcp_tw_reuse`, and
* `net.ipv4.tcp_tw_recycle`.

### Socket lingering

When `close()` is called, any remaining data in the kernel buffers will be sent in the background and the socket will eventually transition to the `TIME-WAIT` state. The application can continue to work immediatly and assume that all data will eventually be safely delivered.

However, __an application can choose to disable this behaviour, known as socket lingering__. 
There are two flavors:

1. In the first one, any remaining data will be discarded and instead of closing the connection with the normal four-packet connection termination sequence, the connection will be closed with a RST (and therefore, the peer will detect an error) and will be immediatly destroyed. No `TIME-WAIT` state in this case.

2. With the second flavor, if there is any data still remaining in the socket send buffer, the process will sleep when calling `close()` until either all the data is sent and acknowledged by the peer or the configured linger timer expires. It is possible for a process to not sleep by setting the socket as non-blocking. In this case, the same process happens in the background. It permits the remaining data to be sent during a configured timeout but if the data is succesfully sent, the normal close sequence is run and you get a `TIME-WAIT` state. And on the other case, you’ll get the connection close with a RST and the remaining data is discarded.

In both cases, disabling socket lingering is not a one-size-fits-all solution. It may be used by some applications like HAProxy or Nginx when it is safe to use from the upper protocol point of view. There are good reasons to not disable it unconditionnaly.

### net.ipv4.tcp_tw_reuse

### net.ipv4.tcp_tw_recycle

## Summary