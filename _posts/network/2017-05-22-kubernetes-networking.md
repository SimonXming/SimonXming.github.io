---
layout: post
title: Kubernetes 网络模型
category: 网络
tags: Kubernetes 网络
keywords: Kubernetes
description:
---

## Docker网络基础

### Linux网络名词解释： 

* 网络的命名空间：Linux在网络栈中引入网络命名空间，将独立的网络协议栈隔离到不同的命令空间中，彼此间无法通信；__docker利用这一特性，实现不容器间的网络隔离。__

* Veth设备对：Veth设备对的引入是为了实现在不同网络命名空间的通信。

* Iptables/Netfilter：Netfilter负责在内核中执行各种挂接的规则(过滤、修改、丢弃等)，运行在内核模式中；Iptables模式是在用户模式下运行的进程，负责协助维护内核中Netfilter的各种规则表；通过二者的配合来实现整个Linux网络协议栈中灵活的数据包处理机制。

* 网桥：网桥是一个二层网络设备,通过网桥可以将linux支持的不同的端口连接起来,并实现类似交换机那样的多对多的通信。

* 路由：Linux系统包含一个完整的路由功能，当IP层在处理数据发送或转发的时候，会使用路由表来决定发往哪里。

### Docker网络实现

* 单机网络模式：Bridge 、Host、Container、None，这里具体就不赘述了

* 多机网络模式：一类是 Docker 在 1.9 版本中引入Libnetwork项目，对跨节点网络的原生支持；一类是通过插件（plugin）方式引入的第三方实现方案，比如 Flannel，Calico 等等

## Kubernetes 网络模型

在 Kubernetes 网络中存在两种IP（Pod IP 和 Service Cluster IP），Pod IP 地址是实际存在于某个网卡(可以是虚拟设备)上的，Service Cluster IP它是一个虚拟IP，是由 kube-proxy 使用 Iptables 规则重新定向到其本地端口，再均衡到后端 Pod 的。下面讲讲 Kubernetes Pod 网络设计模型：

1. 基本原则：
每个Pod都拥有一个独立的IP地址（IP per Pod），而且假定所有的pod都在一个可以直接连通的、扁平的网络空间中。

2. 设计原因：
用户不需要额外考虑如何建立Pod之间的连接，也不需要考虑将容器端口映射到主机端口等问题。

3. 网络要求：
所有的容器都可以在不用NAT的方式下同别的容器通讯；所有节点都可在不用NAT的方式下同所有容器通讯；容器的地址和别人看到的地址是同一个地址。

Kubernetes 网络需要解决下面四类不同的问题：

* 高耦合容器和容器间通信
* Pod 和 Pod 间通信
* Pod 和 Service 间通信
* 外部和内部通信

## Kubernetes 中的 PodIP、ClusterIP 和外部 IP

其中，Kubernetes 中管理主要有三种类型的 IP:Pod IP 、Cluster IP 和 外部 IP。

#### Pod IP
Kubernetes 的最小部署单元是 Pod。利用 Flannel 作为不同 HOST 之间容器互通技术时，由 Flannel 和 etcd 维护了一张节点间的路由表。Flannel 的设计目的就是为集群中的所有节点重新规划 IP 地址的使用规则，从而使得不同节点上的容器能够获得 “同属一个内网” 且” 不重复的”IP 地址，并让属于不同节点上的容器能够直接通过内网 IP 通信。 
每个 Pod 启动时，会自动创建一个镜像为 gcr.io/google_containers/pause:2.0.0 的容器，容器内部与外部的通信经由此容器代理，该容器的 IP 也可以称为 Pod IP。

#### Cluster IP
Pod IP 地址是实际存在于某个网卡 (可以是虚拟设备) 上的，但 Service Cluster IP 就不一样了，没有网络设备为这个地址负责。它是由 kube-proxy 使用 Iptables 规则重新定向到其本地端口，再均衡到后端 Pod 的。

当我们的 Service 被创建时，Kubernetes 给它分配一个地址 10.0.0.1。这个地址从我们启动 API 的 service-cluster-ip-range 参数 (旧版本为 portal_net 参数) 指定的地址池中分配，比如 --service-cluster-ip-range=10.0.0.0/16。假设这个 Service 的端口是 1234。集群内的所有 kube-proxy 都会注意到这个 Service。当 proxy 发现一个新的 service 后，它会在本地节点打开一个任意端口，建相应的 iptables 规则，重定向服务的 IP 和 port 到这个新建的端口，开始接受到达这个服务的连接。

当一个客户端访问这个 service 时，这些 iptable 规则就开始起作用，客户端的流量被重定向到 kube-proxy 为这个 service 打开的端口上，kube-proxy 随机选择一个后端 pod 来服务客户。这个流程如下图所示：

![Cluster IP](http://img.ptcms.csdn.net/article/201506/11/5579419f29d51_middle.jpg?_=8485)

根据 Kubernetes 的网络模型，使用 Service Cluster IP 和 Port 访问 Service 的客户端可以坐落在任意代理节点上。外部要访问 Service，我们就需要给 Service 外部访问 IP。

#### 外部 IP

Service 对象在 Cluster IP range 池中分配到的 IP 只能在内部访问，如果服务作为一个应用程序内部的层次，还是很合适的。如果这个 Service 作为前端服务，准备为集群外的客户提供业务，我们就需要给这个服务提供公共 IP 了。

外部访问者是访问集群代理节点的访问者。为这些访问者提供服务，我们可以在定义 Service 时指定其 spec.publicIPs，一般情况下 publicIP 是代理节点的物理 IP 地址。和先前的 Cluster IP range 上分配到的虚拟的 IP 一样，kube-proxy 同样会为这些 publicIP 提供 Iptables 重定向规则，把流量转发到后端的 Pod 上。有了 publicIP，我们就可以使用 load balancer 等常用的互联网技术来组织外部对服务的访问了。

spec.publicIPs 在新的版本中标记为过时了，代替它的是 spec.type=NodePort，这个类型的 service，系统会给它在集群的各个代理节点上分配一个节点级别的端口，能访问到代理节点的客户端都能访问这个端口，从而访问到服务。

## 模型和动机

Kubernetes 从默认的 Docker 网络模型中脱离出来（Docker 1.8 的网络插件已经很接近）。在一个共享网络命名空间的平面内，目标是一个 pod 一个 IP，每个 pod 都可以和其它物理机或者容器实现跨网络通信。每个 pod 一个 IP 创建了一个简洁的，后向兼容的模型，在这个模型里，包括端口分配，组网，命名，服务发现，负载均衡，应用配置和迁移，pod 都可以被当作虚拟机或者物理主机。

另一方面，动态端口分配，要求同时支持静态端口（如对外部可接入的服务）和动态分配的端口；需要对中心型分配和本地获取动态端口做区分，复杂的调度（应为端口是稀缺资源）对用户来说是非常不方便的；复杂的应用配置，造成用户会被端口冲突、重用和耗尽所困扰；要求非标准的方式来命名（如 consul 和 etcd 而不是 DNS）；需要代理或者重定向才能让程序使用标准的命名和寻址技术（如 web 浏览器）；如果还希望监控整个组成员的变化，并且阻碍容器或 pod 迁移（使用 CRIU），就需要监控和缓存无效的地址和端口变化。其它问题中，NAT 分隔地址空间引入了额外的复杂度，打破了自注册机制。

## 容器到容器

所有容器在一个 pod 中表现为它们在同一个主机上并且不受网络限制。每个 Pod 都有一个全局唯一的 IP,  而且假定所有的 Pod 都在一个可以直接连通的，扁平的网络空间中，Pod 之间可以跨主机通信，按照这种网络原则抽象出来的一个 Pod 对应一个 IP 的设计模型也被称作 IP-per-Pod 模型。

它们可以通过端口在本地实现互相通信。这提供了简单（已知静态端口），安全（端口绑定在 localhost，可以被 pod 中其他容器发现但不会被外部看到），还有性能提升。这同样减少了物理机或虚拟机上非容器化应用的迁移。人们运行应用都堆砌到统一个主机上，它已经解决了如何让端口不冲突和已经安排了如何让客户端去找到它。

这个方法确实减少了一个 pod 中容器的隔离性（端口可能冲突），并且可以没有容器私有化端口，但这些看起来都是未来才会面对的相对比较小的问题。另外，本地化 pod 是容器在 pod 中共享同样的资源（如 volumes，cpu，内存等），所以希望并且可以容忍隔离性的减少。通常情况，用户可以控制哪些容器属于同一个 pod，而不能控制哪些 pod 一起运行在一个主机上。

在 yaml 文件中你可以发现我们设置的 replicas 的数量是 1，但是却发现了两个容器在运行，第二个 image 名为 gcr.io/google_containers/pause:2.0，那么第二个容器从何而来？

执行下图中的命令可以发现：

![container pause](http://upload-images.jianshu.io/upload_images/2231313-a8e217498e2dedd6?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第二个容器和第一个容器共享网络命名空间。第一个容器是 Netowrk Container，它不做任何事情，只是用来接管 Pod 的网络。这样做的好处在于避免容器间的相互依赖，使用一个简单的容器来统一管理网络。


## Pod 到 Pod

因为每个 pod 都有一个 “真”（非机器私有的）的 IP 地址，pods 间可以相互通信，不依赖于代理和转换。Pod 可以使用一个常见的端口号，还可以防止更高层次的服务发现系统像 DNS-SD，Consul 或者 Etcd。

当任何容器调用 ioctl（SIOCGIFADDR）（获取接口的地址），可以看到相同的 IP，任何同级别的容器都可以看到这个 IP，就是说每一个 pod 有自己的 IP 地址，并且这个 IP 地址其他 pods 也可以知道。在容器内外通过生成相同 IP 地址和端口，我们创造了一个非 NAT 模式，扁平化的地址空间。运行ip addr show应该可以看到期待的值。这会让所有现在的命名或发现机制到盒子外面去实现，包括自注册机制和应用分发 IP 地址。我们应该对 pod 间网络通信持乐观态度。在一个 pod 内，容器间更像是通过 volumes（如 tmpfs）或者 IPC 来通信。

这种方式和标准的 Docker 模式有所不同。在 Docker 模式下，每一个容器有一个 IP，IP 是在 172.x.x.x 的网段内，并且只能从 SIOCGIFADDR 看到 172.x.x.x 的地址。如果这些容器连接了另一个容器，那同级别容器会看到这个连接来自于一个不同的 IP 而不是容器自己知道的 IP。简而言之，永远不能自注册同一个容器内的任何东西，因为一个容器不能被自己私有的 IP 获取到。

另一个解决方案我们考虑增加一个寻址层：每个容器一个中心化 pod 的 IP。每个容器有自己本地的 IP 地址，这个 IP 只在 pod 内可见。这种方案可能会让在物理机或虚拟机上的容器化应用移动到 pod 中更容易，但会使实现变得更复杂（如需要每个 pod 一个桥，水平分割的 DNS）。造成额外的地址转换，而且会破话自注册和 IP 分配机制。

像 Docker，端口仍可以向主机节点的接口公开，但这样的需求应该从根本上减少。
