---
layout: post
title: Linux Namespace简介 part-1 - UTS
category: 技术
tags: linux 虚拟化
keywords: linux
description:
---

本文为原文翻译，特此声明。
原文链接：https://blog.jtlebi.fr/2013/12/22/introduction-to-linux-namespaces-part-1-uts/
原文中有部分内容为便于理解，并非直译，可能存在一定的疏漏，因此请参考原文理解。

作为我在OVH的工作的一部分，我着手将Linux Namespace作为一个“尚未公布”产品的一种安全机制。Linux Namespace很强大，但是文档却出奇的匮乏，这着实令人震惊。

你们大多数人也许听说过LXC——LinuX Containers，“Chroot on steroids”。从根本上说，它所做的就是将应用与其他隔离。有点类似于chroot所做的，将一个实际的私有root下的应用隔离，但是LXC进一步考虑了进程。LXC依赖Linux Kernel的3项分离机制：

1. Chroot
2. Cgroups
3. Namespaces

我本可以将这个系列命名为“如何构建你自己的LXC”，从而赢得更高的Google rank，但这样显得过于自大。事实上，LXC所做的远不止隔离，它也带来了模板管理，冻结，以及其他更多的东西。这个系列更多的是科普，而不是重新创造轮子。

在这个系列里，我们将使用最简的C程序一步步启动一个包含更多隔离机制的/bin/bash。

我们开始。

Linux提供容器的过程中，真正有趣的地方是，它没有清晰地提供一个“back-box/magical”的容器解决方案，而是提供一个称为“Namespaces”的个体隔离构件，它是一种全新的方案，在一个又一个release中产生。你也可以在特定的应用中单独使用你所需要的一种。

对于3.12内核，Linux支持6种Namespace：

对于3.12内核，Linux支持6种Namespace：

1. UTS: 主机名(本文)
2. IPC: 进程间通信
3. PID: "chroot"进程树
4. NS: 挂载点，首次登陆Linux
5. NET: 网络接入，包括接口
6. USER: 将本地的虚拟user-id映射到真实的user-id

如下是一个完整的框架，它能从子进程运行/bin/bash。
(不包括错误检测)

```c
// main-0-template.c
#include <signal.h>
#include <unistd.h>
#define STACK_SIZE (1024 * 1024)
static char child_stack[STACK_SIZE];
char* const child_args[] = {
    "/bin/bash",
    NULL
};
int child_main(void* arg) {
    printf(" - World !\n");
    execv(child_args[0], child_args);
    printf("Ooops\n");
    return 1;
}
int main() {
    printf(" - Hello ?\n");
    int child_pid = clone(child_main, child_stack + STACK_SIZE, SIGCHLD, NULL);
    waitpid(child_pid, NULL, 0);
    return 0;
}
```

请注意，在这里我们使用的是“clone”而不是“fork”系统调用。This is where the magin happen。

```shell
jean-tiare@jeantiare-Ubuntu:~/blog$ gcc -Wall main.c -o ns && ./ns
 - Hello ?
 - World !
jean-tiare@jeantiare-Ubuntu:~/blog$ # inside the container
jean-tiare@jeantiare-Ubuntu:~/blog$ exit
jean-tiare@jeantiare-Ubuntu:~/blog$ # outside the container
```

OK，cool。不过假如没有注释，我们很难注意到我们在子/bin/bash。事实上，当我在写这篇文章的时候，曾一度一不小心退出parent shell。

如果能够改变一下，岂不是更加酷炫？比如在主机名中加入0% env vars的把戏？仅仅是普通的Namespace？很简单，我们只要

1. 将“CLONE_NEWUTS”标志加入到clone
2. 从child调用“sethostname”

```c
// main-1-uts.c

// (needs root privileges (or appropriate capabilities))
//[...]
int child_main(void* arg) {
    printf(" - World !\n");
    sethostname("In Namespace", 12);
    execv(child_args[0], child_args);
    printf("Ooops\n");
    return 1;
}
int main() {
    printf(" - Hello ?\n");
    int child_pid = clone(child_main, child_stack+STACK_SIZE,
        CLONE_NEWUTS | SIGCHLD, NULL);
    waitpid(child_pid, NULL, 0);
    return 0;
}
```

在运行一下

```shell
jean-tiare@jeantiare-Ubuntu:~/blog$ gcc -Wall main.c -o ns && sudo ./ns
 - Hello ?
 - World !
root@In Namespace:~/blog$ # inside the container
root@In Namespace:~/blog$ exit
jean-tiare@jeantiare-Ubuntu:~/blog$ # outside the container
```

使用namespace真是太TM简单了：clone，设置适当的“CLONE_NEW*”标志，安装新的env，完成！

你希望更加深入？有兴趣可以阅读一下[excellent LWN article series on namespaces](http://lwn.net/Articles/531114/)。
