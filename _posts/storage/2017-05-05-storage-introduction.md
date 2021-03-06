---
layout: post
title: 存储（文件存储、对象存储、块存储）
category: 存储
tags: linux storage
keywords: storage
description:
---

> 如果我们按存储的业务逻辑分，那么可以分为：键值存储（Key-Value Storage）、文件系统（File System）、数据库（Database，不是很严谨，只是作为所有支持多键值的存储的统称）、消息队列（MQ）等等。按这种分类方法存储的种类是无穷尽的，我曾经说过 “存储即数据结构”，表达的就是这个意思。数据结构无穷尽，存储也会无法穷尽。块存储我们很少把它划分到上面的分类中，因为它是虚拟存储设备，是为 VM 服务的一个东西，并不直接面向用户业务（用户不需要调用块存储设备的 api 来完成业务，当然如果我们把 VM 也认为是一个业务系统的话，块存储也勉强可以作为一种特定的数据结构存在）。对象存储（Object Storage）是键值存储（Key-Value Storage）的特例，它假设 Value 是文件，尺寸范围可以很大（比如七牛可以支持文件大小从 0 字节到 1TB）。如果要对对象存储做进一步分类，我能够想到的就是按实现方案来分类。比如按冗余方案来分，可分为多副本、RAID、纠删码等等；或者我们按一致性方案分，可以分为主从结构和对等结构等。

> 不会有什么存储方案能够一统天下。不同业务系统的场景需求不同，对存储的诉求会不同，选择自然会不同。基于纠删码的存储系统，复杂性远高于三副本的存储系统。从易维护的角度，如果业务对存储成本不敏感，那么人们自然而然会选择三副本的方案。

>                                     -- 许式伟

# 存储

按照这三种接口和其应用场景，很容易了解这三种类型的 IO 特点，括号里代表了它在非分布式情况下的对应：

1. 对象存储（键值数据库）：接口简单，一个对象我们可以看成一个文件，只能全写全读，通常以大文件为主，要求足够的 IO 带宽。

2. 块存储（硬盘）：它的 IO 特点与传统的硬盘是一致的，一个硬盘应该是能面向通用需求的，即能应付大文件读写，也能处理好小文件读写。但是硬盘的特点是容量大，热点明显。因此块存储主要可以应付热点问题。另外，块存储要求的延迟是最低的。

3. 文件存储（文件系统）：支持文件存储的接口的系统设计跟传统本地文件系统如 Ext4 这种的特点和难点是一致的，它比块存储具有更丰富的接口，需要考虑目录、文件属性等支持，实现一个支持并行化的文件存储应该是最困难的。但像 HDFS、GFS 这种自己定义标准的系统，可以通过根据实现来定义接口，会容易一点。

============================================================================

## 【块存储】典型设备：磁盘阵列，硬盘

块存储主要是将裸磁盘空间整个映射给主机使用的，就是说例如磁盘阵列里面有 5 块硬盘（为方便说明，假设每个硬盘 1G），然后可以通过划逻辑盘、做 Raid、或者 LVM（逻辑卷）等种种方式逻辑划分出 N 个逻辑的硬盘。（假设划分完的逻辑盘也是 5 个，每个也是 1G，但是这 5 个 1G 的逻辑盘已经于原来的 5 个物理硬盘意义完全不同了。例如第一个逻辑硬盘 A 里面，可能第一个 200M 是来自物理硬盘 1，第二个 200M 是来自物理硬盘 2，所以逻辑硬盘 A 是由多个物理硬盘逻辑虚构出来的硬盘。）

接着块存储会采用映射的方式将这几个逻辑盘映射给主机，主机上面的操作系统会识别到有 5 块硬盘，但是操作系统是区分不出到底是逻辑还是物理的，它一概就认为只是 5 块裸的物理硬盘而已，跟直接拿一块物理硬盘挂载到操作系统没有区别的，至少操作系统感知上没有区别。此种方式下，操作系统还需要对挂载的裸硬盘进行分区、格式化后，才能使用，与平常主机内置硬盘的方式完全无异。

优点：
1. 这种方式的好处当然是因为通过了 Raid 与 LVM 等手段，对数据提供了保护。
2. 另外也可以将多块廉价的硬盘组合起来，成为一个大容量的逻辑盘对外提供服务，提高了容量。
3. 写入数据的时候，由于是多块磁盘组合出来的逻辑盘，所以几块磁盘可以并行写入的，提升了读写效率。
4. 很多时候块存储采用 SAN 架构组网，传输速率以及封装协议的原因，使得传输速度与读写速率得到提升。

缺点：
1. 采用 SAN 架构组网时，需要额外为主机购买光纤通道卡，还要买光纤交换机，造价成本高。
2. 主机之间的数据无法共享，在服务器不做集群的情况下，块存储裸盘映射给主机，再格式化使用后，对于主机来说相当于本地盘，那么主机 A 的本地盘根本不能给主机 B 去使用，无法共享数据。
3. 不利于不同操作系统主机间的数据共享：另外一个原因是因为操作系统使用不同的文件系统，格式化完之后，不同文件系统间的数据是共享不了的。例如一台装了 WIN7/XP，文件系统是 FAT32/NTFS，而 Linux 是 EXT4，EXT4 是无法识别 NTFS 的文件系统的。就像一只 NTFS 格式的 U 盘，插进 Linux 的笔记本，根本无法识别出来。所以不利于文件共享。

## 【文件存储】典型设备：FTP、NFS 服务器

为了克服上述文件无法共享的问题，所以有了文件存储。

文件存储也有软硬一体化的设备，但是其实普通拿一台服务器 / 笔记本，只要装上合适的操作系统与软件，就可以架设 FTP 与 NFS 服务了，架上该类服务之后的服务器，就是文件存储的一种了。

**主机 A 可以直接对文件存储进行文件的上传下载，与块存储不同，主机 A 是不需要再对文件存储进行格式化的，因为文件管理功能已经由文件存储自己搞定了。**

优点：
1. 造价交低：随便一台机器就可以了，另外普通以太网就可以，根本不需要专用的 SAN 网络，所以造价低。
2. 方便文件共享：例如主机 A（WIN7，NTFS 文件系统），主机 B（Linux，EXT4 文件系统），想互拷一部电影，本来不行。加了个主机 C（NFS 服务器），然后可以先 A 拷到 C，再 C 拷到 B 就 OK 了。（例子比较肤浅，请见谅……）

缺点：
读写速率低，传输速率慢：以太网，上传下载速度较慢，另外所有读写都要 1 台服务器里面的硬盘来承担，相比起磁盘阵列动不动就几十上百块硬盘同时读写，速率慢了许多。

## 【对象存储】典型设备：内置大容量硬盘的分布式服务器

对象存储最常用的方案，就是多台服务器内置大容量硬盘，再装上对象存储软件，然后再额外搞几台服务作为管理节点，安装上对象存储管理软件。管理节点可以管理其他服务器对外提供读写访问功能。

之所以出现了对象存储这种东西，是为了克服块存储与文件存储各自的缺点，发扬它俩各自的优点。简单来说块存储读写快，不利于共享，文件存储读写慢，利于共享。能否弄一个读写快，利 于共享的出来呢。于是就有了对象存储。

首先，一个文件包含了了属性（术语叫 metadata，元数据，例如该文件的大小、修改时间、存储路径等）以及内容（以下简称数据）。

以往像 FAT32 这种文件系统，是直接将一份文件的数据与 metadata 一起存储的，存储过程先将文件按照文件系统的最小块大小来打散（如 4M 的文件，假设文件系统要求一个块 4K，那么就将文件打散成为 1000 个小块），再写进硬盘里面，过程中没有区分数据 / metadata 的。而每个块最后会告知你下一个要读取的块的地址，然后一直这样顺序地按图索骥，最后完成整份文件的所有块的读取。

这种情况下读写速率很慢，因为就算你有 100 个机械手臂在读写，但是由于你只有读取到第一个块，才能知道下一个块在哪里，其实相当于只能有 1 个机械手臂在实际工作。而对象存储则将元数据独立了出来，控制节点叫元数据服务器（服务器 + 对象存储管理软件），里面主要负责存储对象的属性（主要是对象的数据被打散存放到了那几台分布式服务器中的信息），而其他负责存储数据的分布式服务器叫做 OSD，主要负责存储文件的数据部分。当用户访问对象，会先访问元数据服务器，元数据服务器只负责反馈对象存储在哪些 OSD，假设反馈文件 A 存储在 B、C、D 三台 OSD，那么用户就会再次直接访问 3 台 OSD 服务器去读取数据。

这时候由于是 3 台 OSD 同时对外传输数据，所以传输的速度就加快了。当 OSD 服务器数量越多，这种读写速度的提升就越大，通过此种方式，实现了读写快的目的。

另一方面，对象存储软件是有专门的文件系统的，所以 OSD 对外又相当于文件服务器，那么就不存在文件共享方面的困难了，也解决了文件共享方面的问题。
所以对象存储的出现，很好地结合了块存储与文件存储的优点。
最后为什么对象存储兼具块存储与文件存储的好处，还要使用块存储或文件存储呢？

1. 有一类应用是需要存储直接裸盘映射的，例如数据库。因为数据库需要存储裸盘映射给自己后，再根据自己的数据库文件系统来对裸盘进行格式化的，所以是不能够采用其他已经被格式化为某种文件系统的存储的。此类应用更适合使用块存储。
2. 对象存储的成本比起普通的文件存储还是较高，需要购买专门的对象存储软件以及大容量硬盘。如果对数据量要求不是海量，只是为了做文件共享的时候，直接用文件存储的形式好了，性价比高。

===========================================================================

IT 系统按应用存储方式可以分为文件存储、对象存储、块存储三种类型。

* __块存储__，其中块是指物理磁盘上以扇区为基础，一个或几个连续的扇区组成的一段磁盘空间，也叫物理块。块存储简单理解就是一块一块的硬盘，直接挂载在主机上，在主机上能够看到的就是一块块的硬盘以及硬盘分区。从存储架构的角度来讲，块存储分为 DAS 存储（Direct-Attached Storage，直连式存储）、SAN 存储（Storage Area Network，存储区域网络）以及云时代的弹性块存储。
* __文件存储__，用户可在虚拟机或物理机上以文件系统方式，基于 POSIX/CIFS/NFS/FTP 等标准协议进行数据存取。文件系统中有分区，有文件夹，子文件夹，形成一个自上而下的文件结构；文件系统中的文件，可以由应用程序进行打开、修改等操作。物理块与文件系统之间的映射关系是：扇区→物理块→逻辑块（卷）→文件系统。也就是说文件系统中存储的数据最终是写入磁盘中的，文件系统是建立在块存储之上的。从存储架构上来说文件系统分为单机文件系统、网络文件系统（NAS Network Attached Storage，网络附属存储) 以及云时代的分布式文件系统。
* __对象存储__，其中对象是指被封装的文件，应用程序不能直接打开修改它，只能整个文件的读取写入。而且对象存储系统没有像文件系统那样有一个很多层级的文件结构，没有多层次的目录结构，就是简单的 2-3 级结构。对象存储概念来源于互联网时代，所以从存储架构来讲，对象存储都是分布式的。


## 文件存储
* __典型代表__：单机文件系统 FAT,Ext4，网络文件系统 NAS, 分布式文件系统 HDFS
    * __单机文件文件系统__，用于操作系统和应用程序的本地存储。应用直接访问本地文件存储；
    * __网络文件系统__，以太网架构，不同主机之间访问同一套文件系统，实现文件共享。但文件系统是单机的，不能分布式扩展。
    * __分布式文件系统__，不同主机共同访问，文件系统本身可以线性扩展。
* __接口协议__：通常是指支持 POSIX 接口，即传统的文件访问接口方式。分布式存储提供了并行化的能力，如 Ceph 的 CephFS(CephFS 是 Ceph 面向文件存储的接口)，GFS，HDFS 这种非 POSIX 接口的类文件存储接口也归入文件存储类比。
* __应用场景__：丰富的应用场景。
* __读写单位__：文件字节

## 对象存储

* __典型代表__：AWS S3,openstack swift;
* __接口协议__：接口就是简单的 GET、PUT、DEL 和其他扩展。
* __应用场景__：接口简单，一个对象可以看成一个文件，全写全读
* 互联网 / 移动互联网应用的静态内容（视频、图片、文件、软件安装包等）。
    * 存储网站、云盘等
    * __读写单位__：对象（整个文件）
* __与文件系统的差异__：本质区别是接口和数据组织结构的区别
    * __存储接口__：对象存储不支持文件存储中的 fread 和 fwrite 类似的随机位置读写操作，对象存储中一个文件 PUT 到对象存储里以后，如果要读取，只能 GET 整个文件，如果要修改一个对象，只能重新 PUT 一个新的到对象存储里，覆盖之前的对象或者形成一个新的版本。
    * __数据结构组织__：在文件存储系统中文件夹可以一级一级嵌套，对象存储层次关系是固定的，而且只有两到三级，是扁平的。每一级的每个元素，在系统中都有唯一的标识，用户通过这个标识来访问容器或者对象，所以，对象存储提供的是一种 K/V 的访问方式。采用扁平的数据组织结构抛弃了嵌套的文件夹，避免维护庞大的目录树。
* __对象存储的优化空间__
    * 编码的优化问题；主要是指错误数据恢复的编码优化，在恢复时候如何考虑尽可能的减少对网络开销的影响。2 个研究方向：LRC(local recovery codes）编码和减少非必要的数据恢复。
    * 存储中计算 / 存储融合，热门方向，计算的任务放到数据存储的地方进行计算。
    * 特大文件或者特小文件的性能优化问题；

## 块存储

* 块存储是将裸磁盘空间整个映射给主机使用的，形成主机上的一个存储设备。然后通过操作系统进行管控。
* __典型代表__：单机直连式块存储 本地硬盘或者独立 DAS 设备，网络块存储 磁盘阵列，弹性块存储 AWS 的 EBS
    * __DAS（Direct Attach STorage） 直连式块存储__：存储直接连接于主机服务，每一台主机服务器有独立的储存设备，每台主机服务器的储存设备无法互通
    * __SAN(Storage Area Network) 区域网络块存储__：是一种连接外接存储设备和服务器的架构，连接到服务器的存储设备，将被操作系统视为直接连接的存储设备。
    * __弹性块存储__，弹性块存储为物理机或虚拟机提供块级别的分布式的、高速数据存储。
* __接口协议__：接口通常以 QEMU Driver 或者 Kernel Module 的方式存在，如 Sheepdog，AWS 的 EBS，Ceph 的 RBD（RBD 是 Ceph 面向块存储的接口）
* __应用场景__：
    * DAS、SAN 是传统的块存储方式，数据库的陈列存储或主机操作系统的物理存储；
    * 弹性块存储是虚拟存储设备，简单理解就是虚拟网络硬盘，通过卷映射挂到物理机或虚拟机上。
* __读写单位__：应用不直接读取，主机对块存储设备以卷的进行直接映射

## 三种存储方式的差异

* 服务对象差异
    * 块存储直接以卷的方式映射给物理机或虚拟机。不直接面向业务应用。
    * 对象存储、文件存储直接应用于具体的业务应用
* 性能指标差异
    * 块存储，要求的访问时延是 10ms 级的，但 qps（iops）很高
    * 对象存储，文件存储，要求的访问时延是 100ms – 1s 级的，低 qps 的
* 应用场景
    * 块存储，物理主机的本地磁盘、磁盘阵列以及虚拟机的虚拟硬盘。
    * 文件系统，应用场景的较多，但文件系统结构比较复杂，经常要维护一个很深的目录树。
    * 对象存储，应用于单个文件的整体操作场景，把文件整体看做一个独立的对象。

## 统一存储

统一提供文件存储、对象存储和块存储的分布式存储服务。如ceph，sheepdog

## 分布式存储架构规划

### 需求

* 性能需求 运行或在线系统需要高性能。离线或备份数据需要高容量，低价格。数据量大，性能要求不高。
* 可靠性需求。所有的数据都必须是可靠的，绝对不能丢。存储的容忍底线，
* 差异化需求，不同的需求需要用不同的方式来解决。
* 非功能需求，可靠性要求、可用性要求、时延要求、一致性要求、使用模式相关要求（包括请求大小、QPS/IOPS、吞吐）等。