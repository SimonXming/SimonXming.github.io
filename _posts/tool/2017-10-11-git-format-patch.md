---
layout: post
title: git 创建和使用补丁 patch
category: 工具
tags: git patch
keywords: git
description:
---

## git format-patch （推荐）

适用于 git 的 patch，包含 diff 信息，包含提交人，提交时间等 如果 git format-patch 生成的补丁不能打到当前分支，git am会给出提示，并协助你完成打补丁工作

## 对比分支生成 patch

例：从 master checkout 一个新分支修改然后与 master 对比生成 patch。

```shell
$ git format-patch -M master
# -M 选项表示这个patch要和那个分支比对
$ git am 001-xxx.patch（不必重新commit）
```

## 将 commit 打包成 patch

```shell
# 修改代码
$ vi drivers/bluetooth/btusb.c
# 把代码添加到git管理仓库
$ git add .
# 提交修改
$ git commit -m "some message"
# 查看日志,获取到hash
$ git log
# 生成patch
$ git format-patch -s 1bbe3c8c19
```

或者也可以

```shell
$ git format-patch HEAD^ # 最近的1次commit的patch
$ git format-patch HEAD^^ # 最近的2次commit的patch
$ git format-patch HEAD^^^ # 最近的3次commit的patch
$ git format-patch HEAD^^^^ # 最近的4次commit的patch
```

## 测试, 应用 patch

```shell
# 检查patch文件
$ git apply --stat xxx.patch

#查看是否能应用成功
$ git apply --check xxx.patch

# 应用patch
$ git am -s < xxx.patch
```

## git diff

git diff 生成标准的 patch，只包含 diff 信息

git diff 生成的 Patch 兼容性强，可以用git apply --check 查看补丁是否能够干净顺利地应用到当前分支中。
例：从 master checkout 一个新分支修改然后与 master 对比生成 patch。

```shell
$ git diff master > patch
$ git apply xxx.patch
# 需要重新commit
```