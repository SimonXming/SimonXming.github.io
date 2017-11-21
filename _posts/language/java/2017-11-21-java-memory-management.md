---
layout: post
title: java 内存管理
category: 语言
tags: java 基础
keywords: java memory
description:
---

## 堆和非堆内存

按照官方的说法：“Java 虚拟机具有一个堆 (Heap)，堆是运行时数据区域，所有类实例和数组的内存均从此处分配。堆是在 Java 虚拟机启动时创建的。”“在 JVM 中堆之外的内存称为非堆内存 (Non-heap memory)”。

JVM 主要管理两种类型的内存：堆和非堆。

Heap memory|non-heap memory
-----------|---------------
Code Cache|Perm Gen
Eden Space|native heap?(I guess)
Survivor Space|
Tenured Gen|

### 堆内存

Java 虚拟机具有一个堆，堆是运行时数据区域，所有类实例和数组的内存均从此处分配。堆是在 Java 虚拟机启动时创建的。对象的堆内存由称为垃圾回收器的自动内存管理系统回收。

堆的大小可以固定，也可以扩大和缩小。堆的内存不需要是连续空间。

### 非堆内存

Java 虚拟机管理堆之外的内存（称为非堆内存）。

Java 虚拟机具有一个由所有线程共享的方法区。方法区属于非堆内存。它存储每个类结构，如运行时常数池、字段和方法数据，以及方法和构造方法的代码。它是在 Java 虚拟机启动时创建的。

方法区在逻辑上属于堆，但 Java 虚拟机实现可以选择不对其进行回收或压缩。与堆类似，方法区的大小可以固定，也可以扩大和缩小。方法区的内存不需要是连续空间。

除了方法区外，Java 虚拟机实现可能需要用于内部处理或优化的内存，这种内存也是非堆内存。例如，JIT 编译器需要内存来存储从 Java 虚拟机代码转换而来的本机代码，从而获得高性能。

## 几个基本概念

PermGen space：全称是 Permanent Generation space，即永久代。就是说是永久保存的区域, 用于存放 Class 和 Meta 信息，Class 在被 Load 的时候被放入该区域，GC(Garbage Collection) 应该不会对 PermGen space 进行清理，所以如果你的 APP 会 LOAD 很多 CLASS 的话，就很可能出现 PermGen space 错误。

Heap space：存放 Instance。

Java Heap 分为 3 个区，Young 即新生代，Old 即老生代和 Permanent。

Young 保存刚实例化的对象。当该区被填满时，GC 会将对象移到 Old 区。Permanent 区则负责保存反射对象。

* 堆内存分配
    * JVM 初始分配的堆内存由 - Xms 指定，默认是物理内存的 1/64；
    * JVM 最大分配的堆内存由 - Xmx 指定，默认是物理内存的 1/4。
    * 默认空余堆内存小于 40% 时，JVM 就会增大堆直到 - Xmx 的最大限制；
    * 空余堆内存大于 70% 时，JVM 会减少堆直到 - Xms 的最小限制。
    * 因此服务器一般设置 - Xms、-Xmx 相等以避免在每次 GC 后调整堆的大小。
    * 说明：如果 - Xmx 不指定或者指定偏小，应用可能会导致 java.lang.OutOfMemory 错误，此错误来自 JVM，不是 Throwable 的，无法用 try...catch 捕捉。
* 非堆内存分配
    * JVM 使用 - XX:PermSize 设置非堆内存初始值，默认是物理内存的 1/64；
    * 由 XX:MaxPermSize 设置最大非堆内存的大小，默认是物理内存的 1/4。
        * 还有一说：MaxPermSize 缺省值和 - server -client 选项相关，-server 选项下默认 MaxPermSize 为 64m，-client 选项下默认 MaxPermSize 为 32m。这个我没有实验。
    * XX:MaxPermSize 设置过小会导致 java.lang.OutOfMemoryError: PermGen space 就是内存益出。
    * 为什么会内存益出：
        1. 这一部分内存用于存放 Class 和 Meta 的信息，Class 在被 Load 的时候被放入 PermGen space 区域，它和存放 Instance 的 Heap 区域不同。
        2. GC(Garbage Collection) 不会在主程序运行期对 PermGen space 进行清理，所以如果你的 APP 会 LOAD 很多 CLASS 的话, 就很可能出现 PermGen space 错误。
        3. 这种错误常见在 web 服务器对 JSP 进行 pre compile 的时候。

## JVM 内存限制 (最大值)

首先 JVM 内存限制于实际的最大物理内存，假设物理内存无限大的话，JVM 内存的最大值跟操作系统有很大的关系。简单的说就 32 位处理器虽然可控内存空间有 4GB, 但是具体的操作系统会给一个限制，这个限制一般是 2GB-3GB（一般来说 Windows 系统下为 1.5G-2G，Linux 系统下为 2G-3G），而 64bit 以上的处理器就不会有限制了。

为什么有的机器我将 - Xmx 和 - XX:MaxPermSize 都设置为 512M 之后 Eclipse 可以启动，而有些机器无法启动？

通过上面对 JVM 内存管理的介绍我们已经了解到 JVM 内存包含两种：堆内存和非堆内存，另外 JVM 最大内存首先取决于实际的物理内存和操作系统。所以说设置 VM 参数导致程序无法启动主要有以下几种原因：

1. 参数中 - Xms 的值大于 - Xmx，或者 - XX:PermSize 的值大于 - XX:MaxPermSize；
2. -Xmx 的值和 - XX:MaxPermSize 的总和超过了 JVM 内存的最大限制，比如当前操作系统最大内存限制，或者实际的物理内存等等。说到实际物理内存这里需要说明一点的是，如果你的内存是 1024MB，但实际系统中用到的并不可能是 1024MB，因为有一部分被硬件占用了。

如果你有一个双核的 CPU，也许可以尝试这个参数: -XX:+UseParallelGC 让 GC 可以更快的执行。（只是 JDK 5 里对 GC 新增加的参数）

如果你的 WEB APP 下都用了大量的第三方 jar，其大小超过了服务器 jvm 默认的大小，那么就会产生内存益出问题了。解决方法： 设置 MaxPermSize 大小。

* 增加服务器启动的 JVM 参数设置： -Xms128m -Xmx256m -XX:PermSize=128M -XX:MaxNewSize=256m -XX:MaxPermSize=256m
* 如 tomcat，修改 TOMCAT_HOME/bin/catalina.sh，在echo "Using CATALINA_BASE: $CATALINA_BASE"上面加入以下行：JAVA_OPTS="-server -XX:PermSize=64M -XX:MaxPermSize=128m

建议：将相同的第三方 jar 文件移置到 tomcat/shared/lib 目录下，这样可以减少 jar 文档重复占用内存

## JVM 内存设置参数

设置项|  说明
-----|-------
-Xms512m|表示 JVM 初始分配的堆内存大小为 512m（JVM Heap(堆内存) 最小尺寸，初始分配）
-Xmx1024m|JVM 最大允许分配的堆内存大小为 1024m，按需分配（JVM Heap(堆内存) 最大允许的尺寸，按需分配）
-XX:PermSize=512M|JVM 初始分配的非堆内存
-XX:MaxPermSize=1024M|JVM 最大允许分配的非堆内存，按需分配
-XX:NewSize/-XX:MaxNewSize|定义 YOUNG 段的尺寸，NewSize 为 JVM 启动时 YOUNG 的内存大小；MaxNewSize 为最大可占用的 YOUNG 内存大小。
-XX:SurvivorRatio|设置 YOUNG 代中 Survivor 空间和 Eden 空间的比例

### 说明：
1. 如果 - Xmx 不指定或者指定偏小，应用可能会导致 java.lang.OutOfMemory 错误，此错误来自 JVM 不是 Throwable 的，无法用 try...catch 捕捉。
2. PermSize 和 MaxPermSize 指明虚拟机为 java 永久生成对象（Permanate generation）如，class 对象、方法对象这些可反射（reflective）对象分配内存限制，这些内存不包括在 Heap（堆内存）区之中。
3. -XX:MaxPermSize 分配过小会导致：java.lang.OutOfMemoryError: PermGen space。
4. MaxPermSize 缺省值和 - server -client 选项相关：-server 选项下默认 MaxPermSize 为 64m、-client 选项下默认 MaxPermSize 为 32m。

### 申请一块内存的过程

1. JVM 会试图为相关 Java 对象在 Eden 中初始化一块内存区域
2. 当 Eden 空间足够时，内存申请结束。否则到下一步
3. JVM 试图释放在 Eden 中所有不活跃的对象（这属于 1 或更高级的垃圾回收）；释放后若 Eden 空间仍然不足以放入新对象，则试图将部分 Eden 中活跃对象放入 Survivor 区 / OLD 区
4. Survivor 区被用来作为 Eden 及 OLD 的中间交换区域，当 OLD 区空间足够时，Survivor 区的对象会被移到 Old 区，否则会被保留在 Survivor 区
5. 当 OLD 区空间不够时，JVM 会在 OLD 区进行完全的垃圾收集（0 级）
6. 完全垃圾收集后，若 Survivor 及 OLD 区仍然无法存放从 Eden 复制过来的部分对象，导致 JVM 无法在 Eden 区为新对象创建内存区域，则出现”out of memory 错误”


## 内存回收算法

Java 中有四种不同的回收算法，对应的启动参数为:
```shell
–XX:+UseSerialGC
–XX:+UseParallelGC
–XX:+UseParallelOldGC
–XX:+UseConcMarkSweepGC
```

### Serial Collector

大部分平台或者强制 java -client 默认会使用这种。

young generation 算法 = serial

old generation 算法 = serial (mark-sweep-compact)

这种方法的缺点很明显, stop-the-world, 速度慢。服务器应用不推荐使用。

### Parallel Collector

在 linux x64 上默认是这种，其他平台要加 java -server 参数才会默认选用这种。

young = parallel，多个 thread 同时 copy

old = mark-sweep-compact = 1

优点：新生代回收更快。因为系统大部分时间做的 gc 都是新生代的，这样提高了 throughput(cpu 用于非 gc 时间)

缺点：当运行在 8G/16G server 上 old generation live object 太多时候 pause time 过长

### Parallel Compact Collector (ParallelOld)

young = parallel = 2

old = parallel，分成多个独立的单元，如果单元中 live object 少则回收，多则跳过

优点：old old generation 上性能较 parallel 方式有提高

缺点：大部分 server 系统 old generation 内存占用会达到 60%-80%, 没有那么多理想的单元 live object 很少方便迅速回收，同时 compact 方面开销比起 parallel 并没明显减少。

### Concurrent Mark-Sweep(CMS) Collector

young generation = parallel collector = 2

old = cms

同时不做 compact 操作。

优点：pause time 会降低, pause 敏感但 CPU 有空闲的场景需要建议使用策略 4.

缺点：cpu 占用过多，cpu 密集型服务器不适合。另外碎片太多，每个 object 的存储都要通过链表连续跳 n 个地方，空间浪费问题也会增大。

## 内存监控方法

jmap -heap 查看 java 堆（heap）使用情况

```shell
jmap -heap <pid>

using thread-local object allocation.

Parallel GC with 4 thread(s)   #GC 方式

Heap Configuration:  #堆内存初始化配置

MinHeapFreeRatio=40  #对应jvm启动参数-XX:MinHeapFreeRatio设置JVM堆最小空闲比率(default 40)
MaxHeapFreeRatio=70  #对应jvm启动参数 -XX:MaxHeapFreeRatio设置JVM堆最大空闲比率(default 70)
MaxHeapSize=512.0MB  #对应jvm启动参数-XX:MaxHeapSize=设置JVM堆的最大大小
NewSize  = 1.0MB     #对应jvm启动参数-XX:NewSize=设置JVM堆的‘新生代’的默认大小
MaxNewSize =4095MB   #对应jvm启动参数-XX:MaxNewSize=设置JVM堆的‘新生代’的最大大小
OldSize  = 4.0MB     #对应jvm启动参数-XX:OldSize=<value>:设置JVM堆的‘老生代’的大小
NewRatio  = 8        #对应jvm启动参数-XX:NewRatio=:‘新生代’和‘老生代’的大小比率
SurvivorRatio = 8    #对应jvm启动参数-XX:SurvivorRatio=设置年轻代中Eden区与Survivor区的大小比值
PermSize= 16.0MB     #对应jvm启动参数-XX:PermSize=<value>:设置JVM堆的‘永生代’的初始大小
MaxPermSize=64.0MB   #对应jvm启动参数-XX:MaxPermSize=<value>:设置JVM堆的‘永生代’的最大大小

Heap Usage:          #堆内存分步

PS Young Generation

Eden Space:         #Eden区内存分布

capacity = 20381696 (19.4375MB)             #Eden区总容量
used     = 20370032 (19.426376342773438MB)  #Eden区已使用
free     = 11664 (0.0111236572265625MB)     #Eden区剩余容量
99.94277218147106% used                     #Eden区使用比率

From Space:        #其中一个Survivor区的内存分布

capacity = 8519680 (8.125MB)
used     = 32768 (0.03125MB)
free     = 8486912 (8.09375MB)
0.38461538461538464% used

To Space:          #另一个Survivor区的内存分布

capacity = 9306112 (8.875MB)
used     = 0 (0.0MB)
free     = 9306112 (8.875MB)
0.0% used

PS Old Generation  #当前的Old区内存分布

capacity = 366280704 (349.3125MB)
used     = 322179848 (307.25464630126953MB)
free     = 44100856 (42.05785369873047MB)
87.95982001825573% used

PS Perm Generation #当前的 “永生代” 内存分布

capacity = 32243712 (30.75MB)
used     = 28918584 (27.57891082763672MB)
free     = 3325128 (3.1710891723632812MB)
89.68751488662348% used
```