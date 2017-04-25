---
layout: post
title: Docker、Containerd、RunC...：你应该知道的所有
category: 工具
tags: docker 升级
keywords: docker
description:
---

从 Docker 1.11 开始，Docker 容器运行已经不是简单的通过 Docker daemon 来启动，而是集成了 containerd、runC 等多个组件。Docker 服务启动之后，我们也可以看见系统上启动了 dockerd、docker-containerd 等进程，本文主要介绍新版 Docker（1.11 以后）每个部分的功能和作用。

## Docker Daemon

作为 Docker 容器管理的守护进程，Docker Daemon 从最初集成在docker命令中（1.11 版本前），到后来的独立成单独二进制程序（1.11 版本开始），其功能正在逐渐拆分细化，被分配到各个单独的模块中去。从 Docker 服务的启动脚本，也能看见守护进程的逐渐剥离：

在 Docker 1.8 之前，Docker 守护进程启动的命令为：

```shell
docker -d
```
这个阶段，守护进程看上去只是 Docker client 的一个选项。

Docker 1.8 开始，启动命令变成了：

```shell
docker daemon
```
这个阶段，守护进程看上去是docker命令的一个模块。

Docker 1.11 开始，守护进程启动命令变成了：

```shell
dockerd
```

此时已经和 Docker client 分离，独立成一个二进制程序了。

当然，守护进程模块不停的在重构，其基本功能和定位没有变化。和一般的 CS 架构系统一样，守护进程负责和 Docker client 交互，并管理 Docker 镜像、容器。

下面就来介绍下独立分拆出来的其他几个模块。

### Containerd

containerd 是容器技术标准化之后的产物，为了能够兼容 OCI 标准，将容器运行时及其管理功能从 Docker Daemon 剥离。理论上，即使不运行 dockerd，也能够直接通过 containerd 来管理容器。（当然，containerd 本身也只是一个守护进程，容器的实际运行时由后面介绍的 runC 控制。）

最近，Docker 刚刚宣布开源 containerd。从其项目介绍页面可以看出，containerd 主要职责是镜像管理（镜像、元信息等）、容器执行（调用最终运行时组件执行）。

![containerd](http://cdn3.infoqstatic.com/statics_s1_20170411-0445/resource/news/2017/02/Docker-Containerd-RunC/zh/resources/2.png)

containerd 向上为 Docker Daemon 提供了 gRPC 接口，使得 Docker Daemon 屏蔽下面的结构变化，确保原有接口向下兼容。向下通过 containerd-shim 结合 runC，使得引擎可以独立升级，避免之前 Docker Daemon 升级会导致所有容器不可用的问题。

Docker、containerd 和 containerd-shim 之间的关系，可以通过启动一个 Docker 容器，观察进程之间的关联。首先启动一个容器，

```shell
docker run -d busybox sleep 1000
```

然后通过pstree命令查看进程之间的父子关系（其中 20708 是dockerd的 PID）：

```shell
pstree -l -a -A 20708
```
输出结果如下：

```shell
dockerd -H fd:// --storage-driver=overlay2
  |-docker-containe -l unix:///var/run/docker/libcontainerd/docker-containerd.sock --metrics-interval=0 --start-timeout 2m --state-dir /var/run/docker/libcontainerd/containerd --shim docker-containerd-shim --runtime docker-runc
  |   |-docker-containe b9a04a582b66206492d29444b5b7bc6ec9cf1eb83eff580fe43a039ad556e223 /var/run/docker/libcontainerd/b9a04a582b66206492d29444b5b7bc6ec9cf1eb83eff580fe43a039ad556e223 docker-runc
  |   |   |-sleep 1000
```

虽然pstree命令截断了命令，但我们还是能够看出，当 Docker daemon 启动之后，dockerd 和 docker-containerd 进程一直存在。当启动容器之后，docker-containerd 进程（也是这里介绍的 containerd 组件）会创建 docker-containerd-shim 进程，其中的参数 b9a04a582b66206492d29444b5b7bc6ec9cf1eb83eff580fe43a039ad556e223 就是要启动容器的 id。最后 docker-containerd-shim 子进程，已经是实际在容器中运行的进程（既 sleep 1000）。

docker-containerd-shim 另一个参数，是一个和容器相关的目录 `/ var/run/docker/libcontainerd/b9a04a582b66206492d29444b5b7bc6ec9cf1eb83eff580fe43a039ad556e223`，里面的内容有：
```shell
.
├── config.json
├── init-stderr
├── init-stdin
└── init-stdout
```

其中包括了容器配置和标准输入、标准输出、标准错误三个管道文件。

### RunC

OCI 定义了容器运行时标准，runC 是 Docker 按照开放容器格式标准（OCF, Open Container Format）制定的一种具体实现。

runC 是从 Docker 的 libcontainer 中迁移而来的，实现了容器启停、资源隔离等功能。Docker 默认提供了 docker-runc 实现，事实上，通过 containerd 的封装，可以在 Docker Daemon 启动的时候指定 runc 的实现。

我们可以通过启动 Docker Daemon 时增加--add-runtime参数来选择其他的 runC 现。例如：

```shell
docker daemon --add-runtime "custom=/usr/local/bin/my-runc-replacement"
```
下面就让我们看下这几个模块如何工作。

### 举个例子

这里通过 Docker 一些命令，实现不使用 Docker Daemon 直接启动一个镜像，以便了解 Docker Daemon 每个模块的作用。

首先，需要创建容器标准包，这部分实际上由 containerd 的 bundle 模块实现，将 Docker 镜像转换成容器标准包。

```shell
mkdir my_container
cd my_container
mkdir rootfs
docker export $(docker create busybox) | tar -C rootfs -xvf -
```

上述命令将 busybox 镜像解压缩到指定的 rootfs 目录中。如果本地不存在 busybox 镜像，containerd 还会通过 distribution 模块去远程仓库拉取。

现在整个 my_container 目录结构如下：

```shell
$ tree -d my_container/
my_container/
└── rootfs
    ├── bin
    ├── dev
    │   ├── pts
    │   └── shm
    ├── etc
    ├── home
    ├── proc
    ├── root
    ├── sys
    ├── tmp
    ├── usr
    │   └── sbin
    └── var
        ├── spool
        │   └── mail
        └── www
17 directories
```

此时，标准包所需的容器数据已经准备完毕，接下来我们需要创建配置文件：

```shell
docker-runc spec
```
此时会生成一个名为config.json的配置文件，该文件和 Docker 容器的配置文件类似，主要包含容器挂载信息、平台信息、进程信息等容器启动依赖的所有数据。

最后，可以通过runc命令来启动容器：

```shell
runc run busybox
```

注意，runc 必须使用 root 权限启动。

执行之后，我们可以看见容器已经启动：

```shell
localhost my_container # runc run busybox
/ # ps aux
PID   USER     TIME   COMMAND
    1 root       0:00 sh
    9 root       0:00 ps aux
```

此时，事实上已经可以不依赖 Docker 本身，如果系统上安装了runc包，即可运行容器。对于 Gentoo 系统来说，安装app-emulation/runc包即可。

当然，也可以使用 docker-runc 命令来启动容器：

```shell
localhost my_container # docker-runc run busybox
/ # ps aux
PID   USER     TIME   COMMAND
    1 root       0:00 sh
    7 root       0:00 ps aux
```

从这里可以看到标准化的重要性。

### 总结

从 Docker 1.11 之后，Docker Daemon 被分成了多个模块以适应 OCI 标准。拆分之后，结构分成了以下几个部分。

![Docker-Containerd-RunC](http://cdn3.infoqstatic.com/statics_s1_20170411-0445/resource/news/2017/02/Docker-Containerd-RunC/zh/resources/3.png)

其中，containerd 独立负责容器运行时和生命周期（如创建、启动、停止、中止、信号处理、删除等），其他一些如镜像构建、卷管理、日志等由 Docker Daemon 的其他模块处理。

Docker 的模块块拥抱了开放标准，希望通过 OCI 的标准化，容器技术能够有很快的发展。

