---
layout: post
title: Kubernetes 组件
category: 工具
tags: k8s 组件 编排
keywords: k8s
description:
---

# Kubernetes 组件

## Kubernetes 服务暴露组件介绍

从 kubernetes 1.2 版本开始，kubernetes 提供了 Ingress 对象来实现对外暴露服务；到目前为止 kubernetes 总共有三种暴露服务的方式:

* LoadBlancer Service
* NodePort Service
* Ingress

### 1.1、LoadBlancer Service

LoadBlancer Service 是 kubernetes 深度结合云平台的一个组件；当使用 LoadBlancer Service 暴露服务时，实际上是通过向底层云平台申请创建一个负载均衡器来向外暴露服务；目前 LoadBlancer Service 支持的云平台已经相对完善，比如国外的 GCE、DigitalOcean，国内的 阿里云，私有云 Openstack 等等，由于 LoadBlancer Service 深度结合了云平台，所以只能在一些云平台上来使用

### 1.2、NodePort Service

NodePort Service 顾名思义，实质上就是通过在集群的每个 node 上暴露一个端口，然后将这个端口映射到某个具体的 service 来实现的，虽然每个 node 的端口有很多 (0~65535)，但是由于安全性和易用性 (服务多了就乱了，还有端口冲突问题) 实际使用可能并不多

### 1.3、Ingress

Ingress 这个东西是 1.2 后才出现的，通过 Ingress 用户可以实现使用 nginx 等开源的反向代理负载均衡器实现对外暴露服务，以下详细说一下 Ingress，毕竟 traefik 用的就是 Ingress

使用 Ingress 时一般会有三个组件:

* 反向代理负载均衡器
* Ingress Controller
* Ingress

#### 1.3.1、反向代理负载均衡器

反向代理负载均衡器很简单，说白了就是 nginx、apache 什么的；在集群中反向代理负载均衡器可以自由部署，可以使用 Replication Controller、Deployment、DaemonSet 等等，不过个人喜欢以 DaemonSet 的方式部署，感觉比较方便

#### 1.3.2、Ingress Controller

Ingress Controller 实质上可以理解为是个监视器，Ingress Controller 通过不断地跟 kubernetes API 打交道，实时的感知后端 service、pod 等变化，比如新增和减少 pod，service 增加与减少等；当得到这些变化信息后，Ingress Controller 再结合下文的 Ingress 生成配置，然后更新反向代理负载均衡器，并刷新其配置，达到服务发现的作用

#### 1.3.3、Ingress

Ingress 简单理解就是个规则定义；比如说某个域名对应某个 service，即当某个域名的请求进来时转发给某个 service; 这个规则将与 Ingress Controller 结合，然后 Ingress Controller 将其动态写入到负载均衡器配置中，从而实现整体的服务发现和负载均衡

![服务发现和负载均衡](https://mritd.b0.upaiyun.com/markdown/qflqj.jpg)

从上图中可以很清晰的看到，实际上请求进来还是被负载均衡器拦截，比如 nginx，然后 Ingress Controller 通过跟 Ingress 交互得知某个域名对应哪个 service，再通过跟 kubernetes API 交互得知 service 地址等信息；综合以后生成配置文件实时写入负载均衡器，然后负载均衡器 reload 该规则便可实现服务发现，即动态映射

了解了以上内容以后，这也就很好的说明了我为什么喜欢把负载均衡器部署为 Daemon Set；因为无论如何请求首先是被负载均衡器拦截的，所以在每个 node 上都部署一下，同时 hostport 方式监听 80 端口；那么就解决了其他方式部署不确定 负载均衡器在哪的问题，同时访问每个 node 的 80 都能正确解析请求；如果前端再 放个 nginx 就又实现了一层负载均衡

### 1.4、Ingress 详解

Kubernetes 暴露服务的方式目前只有三种：LoadBlancer Service、NodePort Service、Ingress；前两种估计都应该很熟悉，具体的可以参考下 这篇文章；下面详细的唠一下这个 Ingress

#### 1.4.1、Ingress 是个什么玩意

可能从大致印象上 Ingress 就是能利用 Nginx、Haproxy 啥的负载均衡器暴露集群内服务的工具；那么问题来了，集群内服务想要暴露出去面临着几个问题：

#### 1.4.2、Pod 漂移问题

众所周知 Kubernetes 具有强大的副本控制能力，能保证在任意副本 (Pod) 挂掉时自动从其他机器启动一个新的，还可以动态扩容等，总之一句话，这个 Pod 可能在任何时刻出现在任何节点上，也可能在任何时刻死在任何节点上；那么自然随着 Pod 的创建和销毁，Pod IP 肯定会动态变化；那么如何把这个动态的 Pod IP 暴露出去？这里借助于 Kubernetes 的 Service 机制，Service 可以以标签的形式选定一组带有指定标签的 Pod，并监控和自动负载他们的 Pod IP，那么我们向外暴露只暴露 Service IP 就行了；这就是 NodePort 模式：即在每个节点上开起一个端口，然后转发到内部 Service IP 上，如下图所示

![图例](https://mritd.b0.upaiyun.com/markdown/5a1i4.jpg)

#### 1.4.3、端口管理问题

采用 NodePort 方式暴露服务面临一个坑爹的问题是，服务一旦多起来，NodePort 在每个节点上开启的端口会及其庞大，而且难以维护；这时候引出的思考问题是 “能不能使用 Nginx 啥的只监听一个端口，比如 80，然后按照域名向后转发？” 这思路很好，简单的实现就是使用 DaemonSet 在每个 node 上监听 80，然后写好规则，因为 Nginx 外面绑定了宿主机 80 端口 (就像 NodePort)，本身又在集群内，那么向后直接转发到相应 Service IP 就行了，如下图所示

![图例](https://mritd.b0.upaiyun.com/markdown/rrcuu.jpg)

#### 1.4.4、域名分配及动态更新问题

从上面的思路，采用 Nginx 似乎已经解决了问题，但是其实这里面有一个很大缺陷：每次有新服务加入怎么改 Nginx 配置？总不能手动改或者来个 Rolling Update 前端 Nginx Pod 吧？这时候 “伟大而又正直勇敢的” Ingress 登场，如果不算上面的 Nginx，Ingress 只有两大组件：Ingress Controller 和 Ingress

Ingress 这个玩意，简单的理解就是 你原来要改 Nginx 配置，然后配置各种域名对应哪个 Service，现在把这个动作抽象出来，变成一个 Ingress 对象，你可以用 yml 创建，每次不要去改 Nginx 了，直接改 yml 然后创建 / 更新就行了；那么问题来了：”Nginx 咋整？”

Ingress Controller 这东西就是解决 “Nginx 咋整” 的；Ingress Controoler 通过与 Kubernetes API 交互，动态的去感知集群中 Ingress 规则变化，然后读取他，按照他自己模板生成一段 Nginx 配置，再写到 Nginx Pod 里，最后 reload 一下，工作流程如下图

![图例](https://mritd.b0.upaiyun.com/markdown/e5fcy.jpg)

当然在实际应用中，最新版本 Kubernetes 已经将 Nginx 与 Ingress Controller 合并为一个组件，所以 Nginx 无需单独部署，只需要部署 Ingress Controller 即可