---
layout: post
title: 如何写好 Git commit messages
category: 工具
tags: git commit
keywords: git
description:
---


## 导语

任何软件项目都是一个协作项目，它至少需要 2 个开发人员参与，当原始的开发人员将项目开发几个星期或者几个月之后，项目步入正规。不过他们或者后续的开发人员仍然需要经常提交一些代码去修复 bug 或者实现新的 feature。我们经常有这种感受：当一个项目时间过了很久之后，我们对于项目里面的文件和函数功能渐渐淡忘，重新去阅读熟悉这部分代码是很浪费时间并且恼人的一件事。但是这也没法完全避免，我们可以使用一些技巧尽可能减少重新熟悉代码的时间。commit messages 可以满足需要，它也反映了一个开发人员是否是良好的协作者。

编写良好的 Commit messages 可以达到 3 个重要的目的：

* 加快 review 的流程
* 帮助我们编写良好的版本发布日志
* 让之后的维护者了解代码里出现特定变化和 feature 被添加的原因

先来看看一个比较好的例子，感受一下：

![比较好的例子](http://7tszky.com1.z0.glb.clouddn.com/FkeduEr4q7SivzPSQSqLhxa5K7L8)

## Commit messages 的基本语法

```html
<type>(<scope>): <subject>
<BLANK LINE>
<body>
<BLANK LINE>
<footer>
```

Type 表示提交类别，Subject 表示标题行， Body 表示主体描述内容。

Type 的类别说明：

* feat: 添加新特性
* fix: 修复 bug
* docs: 仅仅修改了文档
* style: 仅仅修改了空格、格式缩进、都好等等，不改变代码逻辑
* refactor: 代码重构，没有加新功能或者修复 bug
* perf: 增加代码进行性能测试
* test: 增加测试用例
* chore: 改变构建流程、或者增加依赖库、工具等

格式要求：

```
标题行：50个字符以内，描述主要变更内容
主体内容：更详细的说明文本，建议72个字符以内。 需要描述的信息包括:

* 为什么这个变更是必须的? 它可能是用来修复一个bug，增加一个feature，提升性能、可靠性、稳定性等等
* 他如何解决这个问题? 具体描述解决问题的步骤
* 是否存在副作用、风险? 

尾部：如果需要的化可以添加一个链接到issue地址或者其它文档，或者关闭某个issue。
```

这 3 个问题给代码的变更建立了良好的上下文，可以让其他的代码 review 人员或者项目成员直观的判断修改点是否正确。一条良好的 commit message 也可以让维护者知道这个 patch 是否适合发布到稳定的版本中去。

如果一个 patch 没有回答上面的这几个问题，那么它是无意义的。它会给 review 的人员造成负担，让他们花费大量时间去评审讨论这个 patch 的作用和意义。更糟糕的是，如果一个项目实施 SCM 纪律，则这个 patch 会被拒绝掉，然后开发人员需要花费时间重新编写一个新的 patch。重新编写一个复杂的 patch 代价是巨大的，而把 commit message 写好只会花费几分钟。

## Commit messages 书写建议

* 尽可能多的提交，单个 Commit messages 包含的内容不要过多；
* 标题行以 Fix, Add, Change, Delete 开始，采用命令式的模式。不要使用 Fixed, Added, Changed 等等；
* 始终让第二行是空白，不写任何内容；
* 主体内容注意换行，避免在 gitk 里面滚动条水平滑动；
* 永远不在 git commit 上增加 -m 或 --message= 参数，提交的时候 git commit 即可；
* 如果你是 vim 用户，将下面这行代码加入到~/.vimrc。这会帮助你检查拼写和自动换行。

    ```shell
    autocmd Filetype gitcommit setlocal spell textwidth=72
    ```

## 使用 Commitizen 工具

[Commitizen](https://github.com/commitizen/cz-cli) 可以让你的 commit message 更加规范统一，适合项目团队使用，使用也很简单，使用 npm 安装后，提交代码的时候使用 git cz 去替代以前的 git commit 命令即可。

安装 commitizen：

```shell
$ npm install -g commitizen
```

使用截图：

![Commitizen 使用截图](http://images2015.cnblogs.com/blog/1030776/201702/1030776-20170221140646085-381664858.png)

## 自动生成 Change log

conventional-changelog 是用来从 git 的元数据中生成 Change log 文档的工具，只要你提交的格式满足它定义的标准，此处以 angular 标准为例子。使用它生成的 Change log 包含以下三个部分：

* Bug Fixes Bug 修复的信息
* Features 增加的特性
* BREAKING CHANGES 重大变更

可以参考它生成的文档 CHANGELOG.md，使用如下：

```shell
$ npm install -g conventional-changelog-cli
$ cd my-project
$ conventional-changelog -p angular -i CHANGELOG.md -s
```

这不会覆盖你之前的 CHANGE.md 文档内容，会在这个文件的最上面插入新生成的日志信息。

