---
layout: post
title: Linux 进程控制(前后台切换，signal)
category: 系统
tags: linux process signal
keywords: process
description:
---

### 问题的发现

Happy day@\_@!

为了检查经常出问题的Mysql服务器上连接数及状态，我需要在Centos上安装`net-tools`中包含的`netstat`命令。

```shell
# 查询命令如下
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'  # 查看连接情况

# output
CLOSE_WAIT 32
ESTABLISHED 32
TIME_WAIT 4
SYN_SENT 83
```
After...

```shell
yum install net-tools
```
我转到其他终端继续工作。然后，终端离线了。
当我再次登录尝试安装`net-tools`后。

```shell
已加载插件：fastestmirror
/var/run/yum.pid 已被锁定，PID 为 5599 的另一个程序正在运行。
Another app is currently holding the yum lock; waiting for it to exit...
  另一个应用程序是：yum
    内存： 52 M RSS （449 MB VSZ）
    已启动： Thu Oct 13 12:32:30 2016 - 1:53:40之前
    状态  ：睡眠中，进程ID：5599
```
查看进程状态

```shell
ps -ef|grep yum
# output
5599  5537 S+   /usr/bin/python /usr/bin/yum install net-tools
14597 14404 S+   grep --color=auto yum
```
目测是上次install等待确认`y/d/N`之后进入睡眠状态，然后终端离线了。

```shell
D    Uninterruptible sleep (usually IO)
R    Running or runnable (on run queue)
S    Interruptible sleep (waiting for an event to complete)
T    Stopped, either by a job control signal or because it is being traced.
W    paging (not valid since the 2.6.xx kernel)
X    dead (should never be seen)
Z    Defunct ("zombie") process, terminated but not
     reaped by its parent.

For BSD formats and when the stat keyword is used,additional characters may be displayed:
<    high-priority (not nice to other users)
N    low-priority (nice to other users)
L    has pages locked into memory (for real-time and custom IO)
s    is a session leader
l    is multi-threaded (using CLONE_THREAD, like NPTL pthreads do)
+    is in the foreground process group
```

我当然可以`kill -9 pid`, 但那样就没意思了。为了解决这个问题，我想到了两种解决方案：
1. 恢复到yum install进程的状态，然后手动输入y继续安装(这里引出了jobs, bg, fg和进程前后台切换的问题)
2. 利用linux signal系统使进程在后台继续执行

#### 方案一

```shell
# 查询当前处于后台的进程
jobs -l
# output
[1]+ 15774 运行中               nohup tail -f start_ha.sh &

# 切换进程到前台
fg 1
```
`jobs`命令只能查看当前bash(ppid)进程树下的jobs.继续探索会接触跨ppid进程切换。

### 方案二

根据[stackoverflow上的这篇回答](http://serverfault.com/questions/188936/writing-to-stdin-of-background-process/297095#297095)
给出如下解决方案

```shell
# 方法一: 对于希望启动server时，把stdin与一个fd连接，以便接受stdin
mkfifo /tmp/srv-input
cat > /tmp/srv-input &
echo $! > /tmp/srv-input-cat-pid
cat /tmp/srv-input | myserver &

# 方法二: 对于已启动的服务，可通过pid关联其stdin.
gdb -p PID
call close(0)
call open(0, "/tmp/srv-input", 0600)

echo "command" > /tmp/srv-input
```
