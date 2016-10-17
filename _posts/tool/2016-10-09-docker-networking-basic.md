---
layout: post
title: Docker 网络配置
category: 工具
tags: docker 网络
keywords: mysql
description:
---

![Docker - container and lightweight virtualization](http://blog.opskumu.com/images/docker_networking.png)

Dokcer 通过使用 Linux 桥接提供容器之间的通信，docker0 桥接接口的目的就是方便 Docker 管理。当 Docker daemon 启动时需要做以下操作：

* creates the docker0 bridge if not present(如果 docker0 不存在则创建)
* searches for an IP address range which doesn’t overlap with an existing route(搜索一个与当前路由不冲突的 ip 段)
* picks an IP in the selected range(在确定的范围中选择 ip)
* assigns this IP to the docker0 bridge(绑定 ip 到 docker0)

### 1. Docker 四种网络模式

四种网络模式摘自 [Docker 网络详解及 pipework 源码解读与实践](http://www.infoq.com/cn/articles/docker-network-and-pipework-open-source-explanation-practice)

docker run 创建 Docker 容器时，可以用 --net 选项指定容器的网络模式，Docker 有以下 4 种网络模式：

* host 模式，使用 --net=host 指定。
* container 模式，使用 --net=container:NAME_or_ID 指定。
* none 模式，使用 --net=none 指定。
* bridge 模式，使用 --net=bridge 指定，默认设置。

#### host 模式

如果启动容器的时候使用 host 模式，那么这个容器将不会获得一个独立的 Network Namespace，而是和宿主机共用一个 Network Namespace。容器将不会虚拟出自己的网卡，配置自己的 IP 等，而是使用宿主机的 IP 和端口。

例如，我们在 10.10.101.105/24 的机器上用 host 模式启动一个含有 web 应用的 Docker 容器，监听 tcp 80 端口。当我们在容器中执行任何类似 ifconfig 命令查看网络环境时，看到的都是宿主机上的信息。而外界访问容器中的应用，则直接使用 10.10.101.105:80 即可，不用任何 NAT 转换，就如直接跑在宿主机中一样。但是，容器的其他方面，如文件系统、进程列表等还是和宿主机隔离的。

#### container 模式

这个模式指定新创建的容器和已经存在的一个容器共享一个 Network Namespace，而不是和宿主机共享。新创建的容器不会创建自己的网卡，配置自己的 IP，而是和一个指定的容器共享 IP、端口范围等。同样，两个容器除了网络方面，其他的如文件系统、进程列表等还是隔离的。两个容器的进程可以通过 lo 网卡设备通信。

#### none模式

这个模式和前两个不同。在这种模式下，Docker 容器拥有自己的 Network Namespace，但是，并不为 Docker容器进行任何网络配置。也就是说，这个 Docker 容器没有网卡、IP、路由等信息。需要我们自己为 Docker 容器添加网卡、配置 IP 等。

#### bridge模式
![The Container World | Part 2 Networking](http://blog.opskumu.com/images/vethbridge.png)

bridge 模式是 Docker 默认的网络设置，此模式会为每一个容器分配 Network Namespace、设置 IP 等，并将一个主机上的 Docker 容器连接到一个虚拟网桥上。当 Docker server 启动时，会在主机上创建一个名为 docker0 的虚拟网桥，此主机上启动的 Docker 容器会连接到这个虚拟网桥上。虚拟网桥的工作方式和物理交换机类似，这样主机上的所有容器就通过交换机连在了一个二层网络中。接下来就要为容器分配 IP 了，Docker 会从 RFC1918 所定义的私有 IP 网段中，选择一个和宿主机不同的IP地址和子网分配给 docker0，连接到 docker0 的容器就从这个子网中选择一个未占用的 IP 使用。如一般 Docker 会使用 172.17.0.0/16 这个网段，并将 172.17.42.1/16 分配给 docker0 网桥（在主机上使用 ifconfig 命令是可以看到 docker0 的，可以认为它是网桥的管理接口，在宿主机上作为一块虚拟网卡使用）



[更多内容](https://opskumu.gitbooks.io/docker/content/chapter6.html)
