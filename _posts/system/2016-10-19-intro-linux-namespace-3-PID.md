---
layout: post
title: Linux Namespace简介 part-3 - PID
category: 系统
tags: linux 虚拟化
keywords: linux
description:
---

本文为原文翻译，特此声明。
原文链接：https://blog.jtlebi.fr/2014/01/05/introduction-to-linux-namespaces-part-3-pid/
原文中有部分内容为便于理解，并非直译，可能存在一定的疏漏，因此请参考原文理解。

继上一篇 关于 IPC namespace 的文章 (进程间通信的隔离)，我现在将介绍我个人 (作为系统管理员) 最喜欢的部分：PID namespaces。如果你尚未阅读过之前的 post，我强烈建议你先阅读一遍这个系列的第一篇 post，了解下 linux namespace 隔离机制。

是的，通过这个 namespace，我们将有可能重置 PID 计数，得到你自己的 “1” 进程。这可以被视为在进程标识符 (identifier) 树中的 “chroot”。尤其是当你需要在日常工作中处理 pid，并且为 4 位数(pid) 所困时，这将是极为方便的。

要激活 PID namespace，只需要把 “CLONE_NEWPID” 标记添加到 “clone” 调用。不需要其他额外的步骤。它也能和其他 namespace 组合使用。

一旦激活，子进程 getpid() 的返回结果将会是不变的 “1”。

但是，请等一下！这样岂不是有两个 “1” 进程了？那么进程管理应该怎么办？

事实上，这真的真的很像 “chroot”。也就是说，视角的改变。

* Host: 所有的进程是可见的，全局的 PIDs (init=1, ..., child=xxx, ...)
* Container: 只有 child + descendant(后代) 是可见的，本地 PIDs (child=1, ...)

以下是示例程序

```c
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>
#define STACK_SIZE (1024 * 1024)
// sync primitive
int checkpoint[2];
static char child_stack[STACK_SIZE];
char* const child_args[] = {
  "/bin/bash",
  NULL
};
int child_main(void* arg) {
  char c;
  // init sync primitive
  close(checkpoint[1]);
  // wait...
  read(checkpoint[0], &c, 1);
  printf(" - [%5d] World !\n", getpid());
  sethostname("In Namespace", 12);
  execv(child_args[0], child_args);
  printf("Ooops\n");
  return 1;
}
int main() {
  // init sync primitive
  pipe(checkpoint);
  printf(" - [%5d] Hello ?\n", getpid());
  int child_pid = clone(child_main, child_stack+STACK_SIZE,
      CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWPID | SIGCHLD, NULL);
  // further init here (nothing yet)
  // signal "done"
  close(checkpoint[1]);
  waitpid(child_pid, NULL, 0);
  return 0;
}
```

运行一下

```shell
jean-tiare@jeantiare-Ubuntu:~/blog$ gcc -Wall main-3-pid.c -o ns && sudo ./ns
 - [ 7823] Hello ?
 - [    1] World !
root@In Namespace:~/blog# echo "=> My PID: $$"
=> My PID: 1
root@In Namespace:~/blog# exit
```

正如预期的一样，即使父进程的 PID 是 “7823”，子进程的 PID 是 “1”。如果想试试更好玩的，你可以尝试使用 “kill -KILL 7823”，来终止父进程。准确地说，不会发生任何事情。。。

```shell
jean-tiare@jeantiare-Ubuntu:~/blog$ gcc -Wall main-3-pid.c -o ns && sudo ./ns
 - [ 7823] Hello ?
 - [    1] World !
root@In Namespace:~/blog# kill -KILL 7823
bash: kill: (7823) - No such process
root@In Namespace:~/blog# exit
```

隔离如我们预期的一样工作着。并且，如之前所写的那样，这种行为很类似 “chroot”，意味着从父进程使用 “top” 或 “ps exf”，将会显示子进程和它未映射的 PID。像 “kill”，“cgroups” 以及其他机制一样，这是进程控制最基本的特性。

等等！说到 “top” 和 “ps exf”，我刚从子进程运行了它们，然后发现和父进程一样的内容。你对我撒谎了！

好吧，并不是这样的。这是因为这些工具从真实的 “/proc” 文件系统获取信息，而它目前尚未隔离。而这个正是下一篇文章的目标。

同时，一个简单的工作区可以是这样的：

```shell
# from child
root@In Namespace:~/blog# mkdir -p proc
root@In Namespace:~/blog# mount -t proc proc proc
root@In Namespace:~/blog# ls proc
1          dma          key-users      net            sysvipc
80         dri          kmsg           pagetypeinfo   timer_list
acpi       driver       kpagecount     partitions     timer_stats
asound     execdomains  kpageflags     sched_debug    tty
buddyinfo  fb           latency_stats  schedstat      uptime
bus        filesystems  loadavg        scsi           version
cgroups    fs           locks          self           version_signature
cmdline    interrupts   mdstat         slabinfo       vmallocinfo
consoles   iomem        meminfo        softirqs       vmstat
cpuinfo    ioports      misc           stat           zoneinfo
crypto     irq          modules        swaps
devices    kallsyms     mounts         sys
diskstats  kcore        mtrr           sysrq-trigger
```

所有东西似乎再一次变得合理了。如预期一样，你从 / bin/bash 本身得到了 PID “1”，并通过 “/bin/ls proc” 的到了对应的 “80”。是不是比通常的 / proc 更加 nice？这正是我喜欢它的原因。

如果你尝试从 namespace 直接在 “/proc” 运行这条命令，它将在 child 中可行，但是会 BREAK 你的主 namespace。例子如下：

```shell
jean-tiare@jeantiare-Ubuntu:~/blog$ ps aux
Error, do this: mount -t proc proc /proc
```

这就是 PID namespace 的全部。有了下一篇文章，我们将能够重新挂载 / proc 本身，也就可以修复 “top” 及类似的工具，使之不会破坏 parent namespace(父命名空间)。谢谢阅读！
