---
layout: post
title: docker 存储驱动原理（如何修改已退出容器里的代码）
category: 工具
tags: docker 文件系统
keywords: docker
description:
---

Docker 最开始采用 AUFS 作为文件系统，也得益于 AUFS 分层的概念，实现了多个 Container 可以共享同一个 image。但由于 AUFS 未并入 Linux 内核，且只支持 Ubuntu，考虑到兼容性问题，在 Docker 0.7 版本中引入了存储驱动， 目前，Docker 支持 AUFS、Btrfs、Device mapper、OverlayFS、ZFS 五种存储驱动。就如 Docker 官网上说的，没有单一的驱动适合所有的应用场景，要根据不同的场景选择合适的存储驱动，才能有效的提高 Docker 的性能。如何选择适合的存储驱动，要先了解存储驱动原理才能更好的判断，本文介绍一下 Docker 五种存储驱动原理详解及应用场景及 IO 性能测试的对比。在讲原理前，先讲一下写时复制和写时分配两个技术。

## 一、原理说明

### 写时复制（CoW）

所有驱动都用到的技术——写时复制（CoW）。CoW 就是 copy-on-write，表示只在需要写时才去复制，这个是针对已有文件的修改场景。比如基于一个 image 启动多个 Container，如果为每个 Container 都去分配一个 image 一样的文件系统，那么将会占用大量的磁盘空间。而 CoW 技术可以让所有的容器共享 image 的文件系统，所有数据都从 image 中读取，只有当要对文件进行写操作时，才从 image 里把要写的文件复制到自己的文件系统进行修改。所以无论有多少个容器共享同一个 image，所做的写操作都是对从 image 中复制到自己的文件系统中的复本上进行，并不会修改 image 的源文件，且多个容器操作同一个文件，会在每个容器的文件系统里生成一个复本，每个容器修改的都是自己的复本，相互隔离，相互不影响。使用 CoW 可以有效的提高磁盘的利用率。

### 用时分配（allocate-on-demand）

而写时分配是用在原本没有这个文件的场景，只有在要新写入一个文件时才分配空间，这样可以提高存储资源的利用率。比如启动一个容器，并不会为这个容器预分配一些磁盘空间，而是当有新文件写入时，才按需分配新空间。

### AUFS

AUFS（AnotherUnionFS）是一种 Union FS，是文件级的存储驱动。AUFS 能透明覆盖一或多个现有文件系统的层状文件系统，把多层合并成文件系统的单层表示。简单来说就是支持将不同目录挂载到同一个虚拟文件系统下的文件系统。这种文件系统可以一层一层地叠加修改文件。无论底下有多少层都是只读的，只有最上层的文件系统是可写的。当需要修改一个文件时，AUFS 创建该文件的一个副本，使用 CoW 将文件从只读层复制到可写层进行修改，结果也保存在可写层。在 Docker 中，底下的只读层就是 image，可写层就是 Container。结构如下图所示：

![AUFS](/assets/img/post/20170612/4b4e87b68f711154747e3141289ec935.png)

### Overlay

Overlay 是 Linux 内核 3.18 后支持的，也是一种 Union FS，和 AUFS 的多层不同的是 Overlay 只有两层：一个 upper 文件系统和一个 lower 文件系统，分别代表 Docker 的容器层和镜像层。当需要修改一个文件时，使用 CoW 将文件从只读的 lower 复制到可写的 upper 进行修改，结果也保存在 upper 层。在 Docker 中，底下的只读层就是 image，可写层就是 Container。结构如下图所示：

![Overlay](/assets/img/post/20170612/2a60e3e8527b2f4c64ae74ff0641cc5c.png)

### Device mapper

Device mapper 是 Linux 内核 2.6.9 后支持的，提供的一种从逻辑设备到物理设备的映射框架机制，在该机制下，用户可以很方便的根据自己的需要制定实现存储资源的管理策略。前面讲的 AUFS 和 OverlayFS 都是文件级存储，而 Device mapper 是块级存储，所有的操作都是直接对块进行操作，而不是文件。Device mapper 驱动会先在块设备上创建一个资源池，然后在资源池上创建一个带有文件系统的基本设备，所有镜像都是这个基本设备的快照，而容器则是镜像的快照。所以在容器里看到文件系统是资源池上基本设备的文件系统的快照，并不有为容器分配空间。当要写入一个新文件时，在容器的镜像内为其分配新的块并写入数据，这个叫用时分配。当要修改已有文件时，再使用 CoW 为容器快照分配块空间，将要修改的数据复制到在容器快照中新的块里再进行修改。Device mapper 驱动默认会创建一个 100G 的文件包含镜像和容器。每一个容器被限制在 10G 大小的卷内，可以自己配置调整。结构如下图所示：

![Device mapper](/assets/img/post/20170612/be6f5867d9e5e0be9ac464449b4fc461.png)

### Btrfs

Btrfs 被称为下一代写时复制文件系统，并入 Linux 内核，也是文件级级存储，但可以像 Device mapper 一直接操作底层设备。Btrfs 把文件系统的一部分配置为一个完整的子文件系统，称之为 subvolume 。那么采用 subvolume，一个大的文件系统可以被划分为多个子文件系统，这些子文件系统共享底层的设备空间，在需要磁盘空间时便从底层设备中分配，类似应用程序调用 malloc() 分配内存一样。为了灵活利用设备空间，Btrfs 将磁盘空间划分为多个 chunk 。每个 chunk 可以使用不同的磁盘空间分配策略。比如某些 chunk 只存放 metadata，某些 chunk 只存放数据。这种模型有很多优点，比如 Btrfs 支持动态添加设备。用户在系统中增加新的磁盘之后，可以使用 Btrfs 的命令将该设备添加到文件系统中。Btrfs 把一个大的文件系统当成一个资源池，配置成多个完整的子文件系统，还可以往资源池里加新的子文件系统，而基础镜像则是子文件系统的快照，每个子镜像和容器都有自己的快照，这些快照则都是 subvolume 的快照。

![Btrfs](/assets/img/post/20170612/bc0327cbb936d9756c965e2da3308a5d.png)

当写入一个新文件时，为在容器的快照里为其分配一个新的数据块，文件写在这个空间里，这个叫用时分配。而当要修改已有文件时，使用 CoW 复制分配一个新的原始数据和快照，在这个新分配的空间变更数据，变结束再更新相关的数据结构指向新子文件系统和快照，原来的原始数据和快照没有指针指向，被覆盖。

### ZFS

ZFS 文件系统是一个革命性的全新的文件系统，它从根本上改变了文件系统的管理方式，ZFS 完全抛弃了 “卷管理”，不再创建虚拟的卷，而是把所有设备集中到一个存储池中来进行管理，用 “存储池” 的概念来管理物理存储空间。过去，文件系统都是构建在物理设备之上的。为了管理这些物理设备，并为数据提供冗余，“卷管理” 的概念提供了一个单设备的映像。而 ZFS 创建在虚拟的，被称为 “zpools” 的存储池之上。每个存储池由若干虚拟设备（virtual devices，vdevs）组成。这些虚拟设备可以是原始磁盘，也可能是一个 RAID1 镜像设备，或是非标准 RAID 等级的多磁盘组。于是 zpool 上的文件系统可以使用这些虚拟设备的总存储容量。

![ZFS](/assets/img/post/20170612/6e9101c07b34f359ab1bf54a879944f9.png)

下面看一下在 Docker 里 ZFS 的使用。首先从 zpool 里分配一个 ZFS 文件系统给镜像的基础层，而其他镜像层则是这个 ZFS 文件系统快照的克隆，快照是只读的，而克隆是可写的，当容器启动时则在镜像的最顶层生成一个可写层。如下图所示：

![ZFS-2](/assets/img/post/20170612/0aa8d2c3b88917e36b0e17129941df5b.png)

当要写一个新文件时，使用按需分配，一个新的数据快从 zpool 里生成，新的数据写入这个块，而这个新空间存于容器（ZFS 的克隆）里。
当要修改一个已存在的文件时，使用写时复制，分配一个新空间并把原始数据复制到新空间完成修改。

## 二、存储驱动的对比及适应场景

![存储驱动的对比及适应场景](/assets/img/post/20170612/3b864c7195e377881a58f1622400ac48.png)

### AUFS VS Overlay

AUFS 和 Overlay 都是联合文件系统，但 AUFS 有多层，而 Overlay 只有两层，所以在做写时复制操作时，如果文件比较大且存在比较低的层，则 AUSF 可能会慢一些。而且 Overlay 并入了 linux kernel mainline，AUFS 没有，所以可能会比 AUFS 快。但 Overlay 还太年轻，要谨慎在生产使用。而 AUFS 做为 docker 的第一个存储驱动，已经有很长的历史，比较的稳定，且在大量的生产中实践过，有较强的社区支持。目前开源的 DC/OS 指定使用 Overlay。

### Overlay VS Device mapper

Overlay 是文件级存储，Device mapper 是块级存储，当文件特别大而修改的内容很小，Overlay 不管修改的内容大小都会复制整个文件，对大文件进行修改显示要比小文件要消耗更多的时间，而块级无论是大文件还是小文件都只复制需要修改的块，并不是整个文件，在这种场景下，显然 device mapper 要快一些。因为块级的是直接访问逻辑盘，适合 IO 密集的场景。而对于程序内部复杂，大并发但少 IO 的场景，Overlay 的性能相对要强一些。

### Device mapper VS Btrfs Driver VS ZFS

Device mapper 和 Btrfs 都是直接对块操作，都不支持共享存储，表示当有多个容器读同一个文件时，需要生活多个复本，所以这种存储驱动不适合在高密度容器的 PaaS 平台上使用。而且在很多容器启停的情况下可能会导致磁盘溢出，造成主机不能工作。Device mapper 不建议在生产使用。Btrfs 在 docker build 可以很高效。
ZFS 最初是为拥有大量内存的 Salaris 服务器设计的，所在在使用时对内存会有影响，适合内存大的环境。ZFS 的 COW 使碎片化问题更加严重，对于顺序写生成的大文件，如果以后随机的对其中的一部分进行了更改，那么这个文件在硬盘上的物理地址就变得不再连续，未来的顺序读会变得性能比较差。ZFS 支持多个容器共享一个缓存块，适合 PaaS 和高密度的用户场景。

## 三、IO 性能对比

测试工具：IOzone（是一个文件系统的 benchmark 工具，可以测试不同的操作系统中文件系统的读写性能）
测试场景：从 4K 到 1G 文件的顺序和随机 IO 性能
测试方法：基于不同的存储驱动启动容器，在容器内安装 IOzone，执行命令：

```shell
./iozone -a -n 4k -g 1g -i 0 -i 1 -i 2 -f /root/test.rar -Rb ./iozone.xls
```

测试项的定义和解释

* Write：测试向一个新文件写入的性能。
* Re-write：测试向一个已存在的文件写入的性能。
* Read：测试读一个已存在的文件的性能。
* Re-Read：测试读一个最近读过的文件的性能。
* Random Read：测试读一个文件中的随机偏移量的性能。
* Random Write：测试写一个文件中的随机偏移量的性能。

测试数据对比

Write：
![write](/assets/img/post/20170612/b45e65992d381490f3f5af3930ca5ca1.png)

Re-write:
![write](/assets/img/post/20170612/d15fc94c8f6f35e9af2cd0a68a1aa035.png)

Read：
![write](/assets/img/post/20170612/ec427dd68d1890d9799679e0835f628d.png)

Re-Read：
![write](/assets/img/post/20170612/f336febbac9e809b40d1717b57e9120b.png)

Random Read：
![write](/assets/img/post/20170612/d5772b683b73225b61ab38db7f66bc11.png)

Random Write：
![write](/assets/img/post/20170612/78a973ba7569fe10ff9d651736c32072.png)

通过以上的性能数据可以看到：
* AUFS 在读的方面性能相比 Overlay 要差一些，但在写的方面性能比 Overlay 要好。
* device mapper 在 512M 以上文件的读写性能都非常的差，但在 512M 以下的文件读写性能都比较好。
* btrfs 在 512M 以上的文件读写性能都非常好，但在 512M 以下的文件读写性能相比其他的存储驱动都比较差。
* ZFS 整体的读写性能相比其他的存储驱动都要差一些。 简单的测试了一些数据，对测试出来的数据原理还需要进一步的解析。

# 如何修改已退出容器里的代码

## devicemapper 驱动

### 方法一

```shell
docker inspect --format '\{\{.GraphDriver.Data.DeviceName\}\}' <CONTAINER_NAME>
# 输出
docker-253:0-204988604-cfa868ae50ecc4d5a7ffe56833ca8ebf0258a8462a33872f3b3a95b7aaf18b35
```

### 方法二

```shell
cat $DOCKER_ROOT_PATH/image/aufs/layerdb/mounts/\$(docker inspect --format \{\{.Id\}\} <CONTAINER_NAME>)/mount-id
```

其中 `cfa868ae50ecc4d5a7ffe56833ca8ebf0258a8462a33872f3b3a95b7aaf18b35` 为容器的 writer 的目录在 Host 上的挂载目录

目录路径为
`$DOCKER_ROOT_PATH/devicemapper/mnt/cfa868ae50ecc4d5a7ffe56833ca8ebf0258a8462a33872f3b3a95b7aaf18b35/rootfs/`

## aufs 驱动

```shell
# 获取到 aufs mount-id
cat $DOCKER_ROOT_PATH/image/aufs/layerdb/mounts/\$(docker inspect --format \{\{.Id\}\} <CONTAINER_NAME>)/mount-id

$DOCKER_ROOT_PATH/aufs/mnt/<MOUNT_ID>              # 容器 writer 的挂载目录
```
aufs 参考 [http://docs.docker.com/engine/userguide/storagedriver/aufs-driver/](http://docs.docker.com/engine/userguide/storagedriver/aufs-driver/)
