---
layout: post
title: PAM（Pluggable Authentication Modules）简介
category: 系统
tags: authentication linux
keywords: linux
description:
---

# PAM（Pluggable Authentication Modules）可动态加载验证模块

在 linux 中执行有些程序时，这些程序在执行前首先要对启动它的用户进行认证，符合一定的要求之后才允许执行，例如 login, su 等

在 linux 中进行身份或是状态的验证程序是由 PAM 来进行的，PAM（Pluggable Authentication Modules）可动态加载验证模块，因为可以按需要动态的对验证的内容进行变更，所以可以大大提高验证的灵活性。


![框架结构](http://s3.51cto.com/wyfs02/M00/3A/FE/wKioL1O8rXvxQA8NAAF-Fq6C464335.jpg)

linux 各个发行版中，PAM 使用的验证模块一般存放在 `/lib/x86_64-linux-gnu/security` 目录下，可以使用 ls 命令进行查看本计算机支持哪些验证控制方式，一般的 PAM 模块名字例如 `pam_unix.so`，模块可以随时在这个目录下添加和删除，这不会直接影响程序运行，具体的影响在 PAM 的配置目录下。

PAM 的配置文件一般存放在 `/etc/pam.conf` 文件，或者 `/etc/pam.d/` 目录下。不过现在一般都会存放在 `/etc/pam.d/` 目录下，之下是相对于每个需要被 PAM 控制的程序的独立配置文件。当一个程序的验证方式配置在 `pam.conf` 和 `pam.d/` 下某文件中出现时，以 `pam.d/` 目录下文件为准。

查看某个程序是否支持 PAM，使用命令：

```shell
# cmd 就代表查看的程序名
$ ldd `which cmd` | grep libpam
```

如果包含 libpam 库，那么该程序就支持 PAM 验证。
举个不是特别恰当的例子：PAM 机制就相当于给一个房屋安装防盗门，也就是对要进入这间屋子的人进行控制，不让这间屋子处于一种任何人都可以随便进入的状态。

1. 支持 PAM 验证呢，就是表示这屋子给安装防盗门预留了空位，
2. 不支持 PAM 呢，就是说房子整个就是封闭的，人根本进不了，或者是房子基本没有墙，不需要对出入进行限制。
3. 在 PAM 机制中，PAM 模块就相当于是防盗门上可以安装的各种锁具，这些锁具各有不同的特点和功能，你可以按需要安装。
4. 相同的，在 PAM 中，程序的配置文件就相当于这个防盗门的制作和安装方案，安在什么地方，在支持的各种锁具中选择合适的锁，然后是开这些锁的先后顺序等这些具体使用规范。

PAM 的各种模块是开发人员预先开发好的，而我们要做的是合理的使用这些模块，让它们保护需要保护的程序。所以要关注的是 PAM 的配置文件目录 `/etc/pam.d/`

拿例子说事，以 login 这个登录程序为例子，文件名是 `/etc/pam.d/login`，内容是（其中一部分）：

```shell
auth       optional     pam_faildelay.so    delay=3000000
auth       required     pam_securetty.so
auth       requisite    pam_nologin.so
session    [success=ok ignore=ignore module_unknow=ignore default=bad]    pam_selinux.so close
@include   common-auth
```

从上面可以看出来，配置文件是按行指定的，每一行是一个完整的定义。

一般第一列指定的内容是：module-type，一共就只有 4 种，分别是：

1. auth：对用户身份进行识别，如提示输入密码，判断是 root 否；
2. account：对账号各项属性进行检查，如是否允许登录，是否达到最大用户数；
3. session：定义登录前，及退出后所要进行的操作，如登录连接信息，用户数据的打开和关闭，挂载 fs；
4. password：使用用户信息来更新数据，如修改用户密码。

第二列内容是：control-flag，有很多，不过一般常用的是 4 种，分别是：

1. optional：不进行成功与否的返回，一般返回一个 pam_ignore；
2. required：表示需要返回一个成功值，如果返回失败，不会立刻将失败结果返回，而是继续进行同类型的下一验证，所有此类型的模块都执行完成后，再返回失败；
3. requisite：与 required 类似，但如果此模块返回失败，则立刻返回失败并表示此类型失败；
4. sufficient：如果此模块返回成功，则直接向程序返回成功，表示此类成功，如果失败，也不影响这类型的返回值。

第三列内容是 PAM 模块的存放路径，默认是在 / lib/security / 目录下，如果在此默认路径下，要填写绝对路径。
第四列内容是 PAM 模块参数，这个需要根据所使用的模块来添加。

## PAM 认证原理


PAM 认证一般遵循这样的顺序

Service(服务) -> PAM(配置文件) -> pam_*.so

PAM 认证首先要确定那一项服务，然后加载相应的 PAM 的配置文件 (位于 `/etc/pam.d` 下)，最后调用认证文件 (位于 `/lib/x86_64-linux-gnu/security` 下) 进行安全认证。认证原理图如下图所示：

![原理](http://s3.51cto.com/wyfs02/M01/3A/FE/wKioL1O8rY-C0rOsAADFEPrJxgE391.jpg)

## 示例

* 现在对 linux 系统的登录程序 login 进行针对用户和时间的登录限制
* 使用模块 pam_time.so

1. 检查 login 程序是否支持 PAM

```shell
root@hdp0:~# ldd `which login` | grep libpam
    libpam.so.0 => /lib/libpam.so.0 (0xb76e2000)
    libpam_misc.so.0 => /lib/libpam_misc.so.0 (0xb76de000)
```

2. 在 `/etc/pam.d/` 下修改 login 配置文件
在 login 配置文件中，在 auth 的定义之后，加入 account 的一条验证

```shell
account     requisite     pam_time.so
```

3. pam_time.so 这个模块需要定义另外一个配置文件 time.conf
在 `/etc/security/time.conf` 中定义限制的具体用户和时间

```shell
vim /etc/security/time.conf
login;tty3;fenix;Th2100-2300
# 作用程序是login，作用在tty3上，fenix允许在周四2100-2300间登录
login;!tty3;!fenix;!Th2100-2300
# 作用程序是login，作用在tty3以外，对用户fenix之外产生影响，允许登录时间是周四2100-2300之外
```

4. 对 PAM 的应用是立刻生效，可以切换到其他 tty 上进行测试。

* [Linux 中 pam 认证详解](http://tyjhz.blog.51cto.com/8756882/1436175/)