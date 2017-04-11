---
layout: post
title: Kubernetes 基础
category: 工具
tags: k8s 容器 编排
keywords: k8s
description:
---

![K8s](http://7xnqwp.com1.z0.glb.clouddn.com/kubernetes-structure)

本文将会简单介绍 Kubernetes 的核心概念。因为这些定义可以在 Kubernetes 的文档中找到，所以文章也会避免用大段的枯燥的文字介绍。相反，我们会使用一些图表（其中一些是动画）和示例来解释这些概念。我们发现一些概念（比如 Service）如果没有图表的辅助就很难全面地理解。在合适的地方我们也会提供 Kubernetes 文档的链接以便读者深入学习。
这就开始吧。

## 什么是 Kubernetes？

Kubernetes（k8s）是自动化容器操作的开源平台，这些操作包括部署，调度和节点集群间扩展。如果你曾经用过 Docker 容器技术部署容器，那么可以将 Docker 看成 Kubernetes 内部使用的低级别组件。Kubernetes 不仅仅支持 Docker，还支持 Rocket，这是另一种容器技术。
使用 Kubernetes 可以：

* 自动化容器的部署和复制
* 随时扩展或收缩容器规模
* 将容器组织成组，并且提供容器间的负载均衡
* 很容易地升级应用程序容器的新版本
* 提供容器弹性，如果容器失效就替换它，等等...

实际上，使用 Kubernetes 只需一个部署文件，使用一条命令就可以部署多层容器（前端，后台等）的完整集群：

```shell
$ kubectl create -f single-config-file.yaml
```

kubectl 是和 Kubernetes API 交互的命令行程序。现在介绍一些核心概念。

## 集群

集群是一组节点，这些节点可以是物理服务器或者虚拟机，之上安装了 Kubernetes 平台。下图展示这样的集群。注意该图为了强调核心概念有所简化。这里可以看到一个典型的 Kubernetes 架构图。

![K8s](http://dockerone.com/uploads/article/20151230/d56441427680948fb56a00af57bda690.png)

上图可以看到如下组件，使用特别的图标表示 Service 和 Label：

* Pod
* Container（容器）
* Label(label)（标签）
* Replication Controller（复制控制器）
* Service（enter image description here）（服务）
* Node（节点）
* Kubernetes Master（Kubernetes 主节点）

## Pod

Pod（上图绿色方框）安排在节点上，包含一组容器和卷。同一个 Pod 里的容器共享同一个网络命名空间，可以使用 localhost 互相通信。Pod 是短暂的，不是持续性实体。你可能会有这些问题：

* 如果 Pod 是短暂的，那么我怎么才能持久化容器数据使其能够跨重启而存在呢？ 是的，Kubernetes支持卷的概念，因此可以使用持久化的卷类型。
* 是否手动创建 Pod，如果想要创建同一个容器的多份拷贝，需要一个个分别创建出来么？可以手动创建单个 Pod，但是也可以使用 Replication Controller 使用 Pod 模板创建出多份拷贝，下文会详细介绍。
* 如果 Pod 是短暂的，那么重启时 IP 地址可能会改变，那么怎么才能从前端容器正确可靠地指向后台容器呢？这时可以使用 Service，下文会详细介绍。

## Lable

正如图所示，一些 Pod 有 Label（enter image description here）。一个 Label 是 attach 到 Pod 的一对键 / 值对，用来传递用户定义的属性。比如，你可能创建了一个 "tier" 和 “app” 标签，通过 Label（tier=frontend, app=myapp）来标记前端 Pod 容器，使用 Label（tier=backend, app=myapp）标记后台 Pod。然后可以使用 Selectors 选择带有特定 Label 的 Pod，并且将 Service 或者 Replication Controller 应用到上面。

## Replication Controller

是否手动创建 Pod，如果想要创建同一个容器的多份拷贝，需要一个个分别创建出来么，能否将 Pods 划到逻辑组里？

Replication Controller 确保任意时间都有指定数量的 Pod“副本” 在运行。如果为某个 Pod 创建了 Replication Controller 并且指定 3 个副本，它会创建 3 个 Pod，并且持续监控它们。如果某个 Pod 不响应，那么 Replication Controller 会替换它，保持总数为 3. 如下面的动画所示：

![Replication Controller](http://dockerone.com/uploads/article/20151230/5e2bad1a25e33e2d155da81da1d3a54b.gif)

如果之前不响应的 Pod 恢复了，现在就有 4 个 Pod 了，那么 Replication Controller 会将其中一个终止保持总数为 3。如果在运行中将副本总数改为 5，Replication Controller 会立刻启动 2 个新 Pod，保证总数为 5。还可以按照这样的方式缩小 Pod，这个特性在执行滚动升级时很有用。

当创建 Replication Controller 时，需要指定两个东西：

* Pod 模板：用来创建 Pod 副本的模板
* Label：Replication Controller 需要监控的 Pod 的标签。

现在已经创建了 Pod 的一些副本，那么在这些副本上如何均衡负载呢？我们需要的是 Service。

## Service

如果 Pods 是短暂的，那么重启时 IP 地址可能会改变，怎么才能从前端容器正确可靠地指向后台容器呢？

Service 是定义一系列 Pod 以及访问这些 Pod 的策略的一层抽象。Service 通过 Label 找到 Pod 组。因为 Service 是抽象的，所以在图表里通常看不到它们的存在，这也就让这一概念更难以理解。

现在，假定有 2 个后台 Pod，并且定义后台 Service 的名称为‘backend-service’，lable 选择器为（tier=backend, app=myapp）。backend-service 的 Service 会完成如下两件重要的事情：

* 会为 Service 创建一个本地集群的 DNS 入口，因此前端 Pod 只需要 DNS 查找主机名为 ‘backend-service’，就能够解析出前端应用程序可用的 IP 地址。
* 现在前端已经得到了后台服务的 IP 地址，但是它应该访问 2 个后台 Pod 的哪一个呢？Service 在这 2 个后台 Pod 之间提供透明的负载均衡，会将请求分发给其中的任意一个（如下面的动画所示）。通过每个 Node 上运行的代理（kube-proxy）完成。这里有更多技术细节。

下述动画展示了 Service 的功能。注意该图作了很多简化。如果不进入网络配置，那么达到透明的负载均衡目标所涉及的底层网络和路由相对先进。如果有兴趣，[这里](http://www.dasblinkenlichten.com/kubernetes-101-networking/)有更深入的介绍。

![Service load balancer](http://dockerone.com/uploads/article/20151230/125bbccce0b3bbf42abab0e520d9250b.gif)

有一个特别类型的 Kubernetes Service，称为'LoadBalancer'，作为外部负载均衡器使用，在一定数量的 Pod 之间均衡流量。比如，对于负载均衡 Web 流量很有用。

## Node

节点（上图橘色方框）是物理或者虚拟机器，作为 Kubernetes worker，通常称为 Minion。每个节点都运行如下 Kubernetes 关键组件：

* Kubelet：是主节点代理。
* Kube-proxy：Service 使用其将链接路由到 Pod，如上文所述。
* Docker 或 Rocket：Kubernetes 使用的容器技术来创建容器。

## Kubernetes Master

集群拥有一个 Kubernetes Master（紫色方框）。Kubernetes Master 提供集群的独特视角，并且拥有一系列组件，比如 Kubernetes API Server。API Server 提供可以用来和集群交互的 REST 端点。master 节点包括用来创建和复制 Pod 的 Replication Controller。
