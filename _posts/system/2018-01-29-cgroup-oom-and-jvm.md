---
layout: post
title: cgroup OOM and JVM
category: 系统
tags: memory jvm docker
keywords: memory
description:
---

- [Linux cgroup](#linux-cgroup)  
    - [Terminology](#terminology)  
    - [cgroup and OOM-killer](#cgroup-and-oom-killer)  
    - [Kernel Memory Extension in cgroup](#kernel-memory-extension-in-cgroup)
    - [why free and top not work in a Linux container?](#why-free-and-top-not-work-in-a-linux-container?)
- [OOM](#oom)  
    - [OOM in k8s](#oom-in-k8s)  
    - [OOM in docker](#oom-in-docker)  
    - [OOM-killer in linux](#oom-killer-in-linux)  
- [JVM memory](#jvm-memory)  
    - [What is RSS?](#what-is-rss?)
    - [What RSS means in java application?](#what-rss-means-in-java-application?)
    - [Why does docker stats info differ from the ps data?](#why-does-docker-stats-info-differ-from-the-ps-data?)

# Linux cgroup

cgroups 可以限制、记录、隔离进程组所使用的物理资源（包括：CPU、memory、IO 等），为容器实现虚拟化提供了基本保证，是构建 Docker 等一系列虚拟化管理工具的基石。

对开发者来说，cgroups 有如下四个有趣的特点：

1. cgroups 的 API 以一个伪文件系统的方式实现，即用户可以通过文件操作实现 cgroups 的组织管理
2. cgroups 的组织管理操作单元可以细粒度到线程级别，用户态代码也可以针对系统分配的资源创建和销毁 cgroups，从而实现资源再分配和管理
3. 所有资源管理的功能都以 "subsystem（子系统）" 的方式实现，接口统一
4. 子进程创建之初与其父进程处于同一个 cgroups 的控制组

本质上来说，cgroups 是内核附加在程序上的一系列钩子（hooks），通过程序运行时对资源的调度触发相应的钩子以达到资源追踪和限制的目的。

cgroup 有 v1, v2 两个版本。(http://man7.org/linux/man-pages/man7/cgroups.7.html)

### cgroup mission

实现 cgroups 的主要目的是为不同用户层面的资源管理，提供一个统一化的接口。从单个进程的资源控制到操作系统层面的虚拟化。Cgroups 提供了以下四大功能 {![参照自：http://en.wikipedia.org/wiki/Cgroups]}。

* 资源限制（Resource Limitation）：cgroups 可以对进程组使用的资源总额进行限制。如设定应用运行时使用内存的上限，一旦超过这个配额就发出 OOM（Out of Memory）
* 优先级分配（Prioritization）：通过分配的 CPU 时间片数量及硬盘 IO 带宽大小，实际上就相当于控制了进程运行的优先级
* 资源统计（Accounting）： cgroups 可以统计系统的资源使用量，如 CPU 使用时长、内存用量等等，这个功能非常适用于计费
* 进程控制（Control）：cgroups 可以对进程组执行挂起、恢复等操作

### Terminology

* task（任务）：cgroups 的术语中，task 就表示系统的一个进程。
* cgroup（控制组）：cgroups 中的资源控制都以 cgroup 为单位实现。cgroup 表示按某种资源控制标准划分而成的任务组，包含一个或多个子系统。一个任务可以加入某个 cgroup，也可以从某个 cgroup 迁移到另外一个 cgroup。
* subsystem（子系统）：cgroups 中的 subsystem 就是一个资源调度控制器（Resource Controller）。比如 CPU 子系统可以控制 CPU 时间分配，内存子系统可以限制 cgroup 内存使用量。
* hierarchy（层级树）：hierarchy 由一系列 cgroup 以一个树状结构排列而成，每个 hierarchy 通过绑定对应的 subsystem 进行资源调度。hierarchy 中的 cgroup 节点可以包含零或多个子节点，子节点继承父节点的属性。整个系统可以有多个 hierarchy。

由此可见，cgroup 对资源的管理是一个树形结构，类似进程。

* 相同点 - 分层结构，子进程/子cgroup 继承父进程/父cgroup
* 不同点 - 进程是一个单根树状结构 (pid=0 为根)，而 cgroup 整体来看是一个多树的森林结构 (hierarchy 为根)

```shell
/cgroup/
├── blkio                           <--------------- hierarchy/root cgroup
│   ├── blkio.io_merged             <--------------- subsystem parameter
... ...
│   ├── blkio.weight
│   ├── blkio.weight_device
│   ├── cgroup.event_control
│   ├── cgroup.procs
│   ├── lxc                         <--------------- cgroup
│   │   ├── blkio.io_merged         <--------------- subsystem parameter
│   │   ├── blkio.io_queued
... ... ...
│   │   └── tasks                   <--------------- task list
│   ├── notify_on_release
│   ├── release_agent
│   └── tasks
...
```

### cgroup 操作准则

1. 一个 hierarchy 可以有多个 subsystem (mount 的时候 hierarchy 可以 attach 多个 subsystem)
2. 一个已经被挂载的 subsystem 只能被再次挂载在一个空的 hierarchy 上 (已经 mount 一个 subsystem 的 hierarchy 不能挂载一个已经被其它 hierarchy 挂载的 subsystem)
3. 每个 task 只能在一同个 hierarchy 的唯一一个 cgroup 里 (不能在同一个 hierarchy 下有超过一个 cgroup 的 tasks 里同时有这个进程的 pid)
4. 子进程在被 fork 出时自动继承父进程所在 cgroup，但是 fork 之后就可以按需调整到其他 cgroup
5. 限制一个 task 的唯一方法就是将其加入到一个 cgroup 的 task 里
6. 多个 subsystem 可以挂载到一个 hierarchy 里, 然后通过不同的 cgroup 中的 subsystem 参数来对不同的 task 进行限额
7. 如果一个 hierarchy 有太多 subsystem，可以考虑重构 - 将 subsystem 挂到独立的 hierarchy; 相应的, 可以将多个 hierarchy 合并成一个 hierarchy
8. 因为可以只挂载少量 subsystem, 可以实现只对 task 单个方面的限额; 同时一个 task 可以被加到多个 hierarchy 中，从而实现对多个资源的控制

* [参考文档](http://tiewei.github.io/devops/howto-use-cgroup/)

### cgroup and OOM-killer

在 cgroup 资源限制的进程中，超出 cgroup 限制的资源申请会失败，然后导致 kernel OOM-killer 在 cgroup 中的选择一个进程释放资源。

* [cgroup Memory Resource Controller](https://www.kernel.org/doc/Documentation/cgroup-v1/memory.txt)

> Reclaim
> 
> Each cgroup maintains a per cgroup LRU which has the same structure as
> global VM. When a cgroup goes over its limit, we first try
> to reclaim memory from the cgroup so as to make space for the new
> pages that the cgroup has touched. If the reclaim is unsuccessful,
> an OOM routine is invoked to select and kill the bulkiest task in the
> cgroup. (See 10. OOM Control below.)
> 
> The reclaim algorithm has not been modified for cgroups, except that
> pages that are selected for reclaiming come from the per-cgroup LRU
> list.
> 
> NOTE: Reclaim does not work for the root cgroup, since we cannot set any
> limits on the root cgroup.
> 
> Note2: When panic_on_oom is set to "2", the whole system will panic.
> 
> When oom event notifier is registered, event will be delivered.
> (See oom_control section)

### Kernel Memory Extension in cgroup

> 2.7 Kernel Memory Extension (CONFIG_MEMCG_KMEM)
> 
> With the Kernel memory extension, the Memory Controller is able to limit
> the amount of kernel memory used by the system. Kernel memory is fundamentally
> different than user memory, since it can't be swapped out, which makes it
> possible to DoS the system by consuming too much of this precious resource.
> 
> Kernel memory accounting is enabled for all memory cgroups by default. But
> it can be disabled system-wide by passing `cgroup.memory=nokmem` to the kernel
> at boot time. In this case, kernel memory will not be accounted at all.
> 
> Kernel memory limits are not imposed for the root cgroup. Usage for the root
> cgroup may or may not be accounted. The memory used is accumulated into
> memory.kmem.usage_in_bytes, or in a separate counter when it makes sense.
> (currently only for tcp).
> The main "kmem" counter is fed into the main counter, so kmem charges will
> also be visible from the user counter.
> 
> Currently no soft limit is implemented for kernel memory. It is future work
> to trigger slab reclaim when those limits are reached.
> 
> 2.7.1 Current Kernel Memory resources accounted
> 
> * stack pages: every process consumes some stack pages. By accounting into
> kernel memory, we prevent new processes from being created when the kernel
> memory usage is too high.
> 
> * slab pages: pages allocated by the SLAB or SLUB allocator are tracked. A copy
> of each kmem_cache is created every time the cache is touched by the first time
> from inside the memcg. The creation is done lazily, so some objects can still be
> skipped while the cache is being created. All objects in a slab page should
> belong to the same memcg. This only fails to hold when a task is migrated to a
> different memcg during the page allocation by the cache.
> 
> * sockets memory pressure: some sockets protocols have memory pressure
> thresholds. The Memory Controller allows them to be controlled individually
> per cgroup, instead of globally.
> 
> * tcp memory pressure: sockets memory pressure for the tcp protocol.
>
> 2.7.2 Common use cases
> 
> Because the "kmem" counter is fed to the main user counter, kernel memory can
> never be limited completely independently of user memory. Say "U" is the user
> limit, and "K" the kernel limit. There are three possible ways limits can be set:
> 
> * U != 0, K = unlimited:  
>    This is the standard memcg limitation mechanism already present before kmem
>    accounting. Kernel memory is completely ignored.
>
> * U != 0, K < U:  
>    Kernel memory is a subset of the user memory. This setup is useful in
>    deployments where the total amount of memory per-cgroup is overcommited.
>    Overcommiting kernel memory limits is definitely not recommended, since the
>    box can still run out of non-reclaimable memory.
>    In this case, the admin could set up K so that the sum of all groups is
>    never greater than the total memory, and freely set U at the cost of his
>    QoS.
>    WARNING: In the current implementation, memory reclaim will NOT be
>    triggered for a cgroup when it hits K while staying below U, which makes
>    this setup impractical.
>
> * U != 0, K >= U:  
>    Since kmem charges will also be fed to the user counter and reclaim will be
>    triggered for the cgroup for both kinds of memory. This setup gives the
>    admin a unified view of memory, and it is also useful for people who just
>    want to track kernel memory usage.

### why free and top not work in a Linux container?

Most of the Linux tools providing system resource metrics were created before cgroups even existed (e.g.: free and top, both from procps). They usually read memory metrics from the proc filesystem: /proc/meminfo, /proc/vmstat, /proc/PID/smaps and others.

Unfortunately /proc/meminfo, /proc/vmstat and friends are not containerized. Meaning that they are not cgroup-aware. They will always display memory numbers from the host system (physical or virtual machine) as a whole.

# OOM

## OOM in k8s

#### why OOM happend in k8s ?

如果节点先发生系统 OOM（out-of-memory）事件，然后 kubelet 能够释放内存，则节点依赖于 [kernel oom_killer](https://lwn.net/Articles/391222/) 进行响应。

kubelet 会基于 POD 的 quality of service 为每个 container 设置一个 `oom_score_adj` 值。

Quality of Service|oom_score_adj
------------------|---------
Guaranteed|-998
BestEffort|1000
Burstable|min(max(2, 1000 - (1000 * memoryRequestBytes) / machineMemoryCapacityBytes), 999)

如果在节点发生系统 OOM（out-of-memory）事件之前，kubelet 无法释放内存，`kernel oom_killer` 会基于节点内存使用率的百分比计算出一个 `oom_score` 值, 然后把这个值和 `oom_score_adj` 相加，然后得出一个对 container 有效的 `oom_score` 值，`oom_killer` 就会杀掉得分最高的 container.

#### k8s 回收资源的方式

1. Kubelet 通过 OOM Killer 来回收资源的缺点

    * System OOM events 本来就是对资源敏感的，它会 stall 这个 Node 直到完成了 OOM Killing Process。
    * 当 OOM Killer 干掉某些 containers 之后，kubernetes Scheduler 可能很快又会调度一个新的 Pod 到该 Node 上或者 container 直接在 node 上 restart，马上又会触发该 Node 上的 OOM Killer 启动 OOM Killing Process

2. Kubelet Eviction Policy 的工作机制  
    （略）

#### 总结

* 如果不启动 Kubelet Eviction Policy，kubelet 使用 `kernel oom_killer` 进行资源紧张节点上的资源释放。但是会通过 k8s 定义的 `oom_score_adj` 值来影响 `oom_killer` 的行为。
* 如果启动 Kubelet Eviction Policy，可以避免 `kernel oom_killer` 导致的 container 意外 kill 等一些缺点。

## OOM in docker

docker daemon 主机内存资源不足时，会使用 `kernel oom_killer` 机制，按照 OOM priority 选取进程进行 kill.

docker 提供了一系列参数控制资源相关的

参数                 |描述  |mount point
--------------------|-----|-----------
--memory            |The maximum amount of memory the container can use.|`/sys/fs/cgroup/memory/memory.limit_in_bytes`
--memory-swap       |The amount of memory this container is allowed to swap to disk.|`/sys/fs/cgroup/memory/memory.memsw.limit_in_bytes`
--memory-swappiness |By default, the host kernel can swap out a percentage of anonymous pages used by a container. You can set --memory-swappiness to a value between 0 and 100|`/sys/fs/cgroup/memory/memory.swappiness`
--memory-reservation|Allows you to specify a soft limit smaller than --memory which is activated when Docker detects contention or low memory on the host machine.|`/sys/fs/cgroup/memory/memory.soft_limit_in_bytes`
--kernel-memory     |The maximum amount of kernel memory the container can use. |`/sys/fs/cgroup/memory/memory.kmem.limit_in_bytes`
--oom-kill-disable  |Only disable the OOM killer on containers where you have also set the -m/--memory option.|`/sys/fs/cgroup/memory/memory.oom_control`

* [官方文档](https://docs.docker.com/engine/admin/resource_constraints/)

## OOM-killer in linux

Linux 下有一种 OOM KILLER 的机制，它会在系统内存耗尽的情况下，启用自己算法有选择性的杀掉一些进程。

上面的描述中说明了在 Linux 中当 malloc 返回的是非空时，并不代表有可以使用的内存空间。Linux 系统允许程序申请比系统可用内存更多的内存空间，这个特性叫做 overcommit 特性，这样做可能是为了系统的优化，因为不是所有的程序申请了内存就会立刻使用，当真正的使用时，系统可能已经回收了一些内存。但是，当你使用时 Linux 系统没有内存可以使用时，OOM Killer 就会出来让一些进程退出。

Linux 下有 3 种 Overcommit 的策略（参考内核文档： Documentation/vm/overcommit-accounting ），可以在 /proc/sys/vm/overcommit_memory 配置（可以取 0,1 和 2 三个值，默认是 0）。

0. 启发式策略 ，比较严重的 Overcommit 将不能得逞，比如你突然申请了 128TB 的内存。而轻微的 overcommit 将被允许。另外，root 能 Overcommit 的值比普通用户要稍微多。
1. 永远允许 overcommit ，这种策略适合那些不能承受内存分配失败的应用，比如某些科学计算应用。
2. 永远禁止 overcommit ，在这个情况下，系统所能分配的内存不会超过 swap+RAM * 系数 （/proc/sys/vm/overcmmit_ratio，默认 50%，你可以调整），如果这么多资源已经用光，那么后面任何尝试申请内存的行为都会返回错误，这通常意味着此时没法运行任何新程序。

# JVM memory

* [java 中堆内存和栈内存详解](http://blog.comprehend.in/2018/01/25/java-heap-stack-memory-allocation.html)
* [java 内存管理](http://blog.comprehend.in/2017/11/21/java-memory-management.html)

## What is RSS?

Resident Set Size (RSS) is the amount of physical memory currently allocated and used by a process (without swapped out pages). It includes the code, data and shared libraries (which are counted in every process which uses them)

## What RSS means in java application?

Theoretically, in case of a java application

```shell
RSS = Heap size + MetaSpace + OffHeap size
```

where OffHeap consists of thread stacks, direct buffers, mapped files (libraries and jars) and JVM code itself.

## Why does docker stats info differ from the ps data?

Answer for the question is very simple - Docker has a bug (or a feature - depends on your mood): it includes file caches into the total memory usage info. So, we can just avoid this metric and use ps info about RSS.