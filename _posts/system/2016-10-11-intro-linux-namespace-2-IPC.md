---
layout: post
title: Linux Namespace简介 part-2 - IPC
category: 系统
tags: linux 虚拟化
keywords: linux
description:
---

本文为原文翻译，特此声明。
原文链接：https://blog.jtlebi.fr/2013/12/22/introduction-to-linux-namespaces-part-2-ipc/
原文中有部分内容为便于理解，并非直译，可能存在一定的疏漏，因此请参考原文理解。

继上一篇 关于UTS namespace的文章(hostname隔离)，我们将更进一步，观察更多面向安全的namespace：IPC，进程间通信。如果你尚未阅读UTS部分，我强烈建议你先阅读一遍这个系列的第一篇post，了解下linux namespace隔离机制。

要激活IPC namespace，只需要把“CLONE_NEWPIC”标记添加到“clone”调用。不需要其他额外的步骤。它也能和其他namespace组合使用。

一旦激活，你将能够如同往常一样创建IPC，甚至是命名一个，而不会有与任何其他应用冲突的风险。

但是，请先等一下！我的“父进程”和“子进程”现在不是隔离的吗？那么如果我需要让它们通信，应该如何解决？

好问题。一个常规的方案是，在child全权控制之前，在parent中增加一些额外的步骤。幸运的是，并非所有的东西都是隔离的，clone和它的parent分享内存空间，因此你依旧可以使用：

* 信号(signal)
* poll内存
* 套接字(sockets)
* 使用文件(file)和文件描述符(fd)

由于上下文(context)的改变，使用signal也许不是最可行的方案。而使用poll memory来做通信则太TM低效了！

如果你并不计划完全隔离网络栈，你可以使用sockets，而文件系统也是类似。但是，在这个系列的例子里，我们要做的事是很清晰的——一步步隔离一切。

一个鲜有人知/很少使用的方案是监视一对pipe上的事件。事实上，这是Lennart Poettering在Systemd's "nspawn"中使用过的技术。我会在这里引入这种极其强大的技术。这也正是我们将在下一篇文章中所依赖的技术。

首先，我们需要初始化一对pipe。让我们称之为“检查点”。

```c
// checkpoint-global-init.c

// required headers:
#include <unistd.h>
// global status:
int checkpoint[2];
// [parent] init:
pipe(checkpoint);
```
这里的思想是，在parent中触发一个“close”事件，并且在child的读取端等待“EOF”被接收。当EOF被接收后，所有的写fd必须被关闭，这是我们需要理解的最重要的东西。因此，在child等待前，应该先关闭自己的写fd的拷贝。

```c
// checkpoint-child-init.c

// required headers:
#include <unistd.h>
// [child] init:
close(checkpoint[1]);
```
现在，“信号”是很直接的：

* 在parent中关闭写fd
* 在child中等待EOF

```c
// checkpoint-signal.c

// required headers:
#include <unistd.h>
// [child] wait:
char c; // stub char
read(checkpoint[0], &c, 1);
// [parent] signal ready code:
close(checkpoint[1]);
```
如果我们将它和第一个例子中的UTS namespace放在一起，将是如下样子：

```c
// main-2-ipc.c
// checkpoint-signal.c

// required headers:
#include <unistd.h>
// [child] wait:
char c; // stub char
read(checkpoint[0], &c, 1);
// [parent] signal ready code:
close(checkpoint[1]);
```
如果我们将它和第一个例子中的UTS namespace放在一起，将是如下样子：

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
  printf(" - World !\n");
  sethostname("In Namespace", 12);
  execv(child_args[0], child_args);
  printf("Ooops\n");
  return 1;
}
int main() {
  // init sync primitive
  pipe(checkpoint);
  printf(" - Hello ?\n");
  int child_pid = clone(child_main, child_stack+STACK_SIZE,
      CLONE_NEWUTS | CLONE_NEWIPC | SIGCHLD, NULL);
  // some damn long init job
  sleep(4);
  // signal "done"
  close(checkpoint[1]);
  waitpid(child_pid, NULL, 0);
  return 0;
}
```

由于需要高级特性，我们需要root或者对等的权限来运行这段代码。很显然，没有必要在这个例子中保留“CLONE_NEWUTS”。我保留它仅仅是为了展示多个namespace也许可以一起使用。

这就是IPC的所有东西，它本身并不复杂。正如我们接下来要做的，它只有在parent/child同步时才变得复杂，而这正需要“pipe”技术来作为一个方便的解决方案。产品中已经使用了这个技术。

下一篇，是我(作为系统管理员)最喜欢的部分：PID namespaces。
