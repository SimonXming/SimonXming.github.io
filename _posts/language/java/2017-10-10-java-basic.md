---
layout: post
title: java 基础知识
category: 语言
tags: java 基础
keywords: java classloader
description:
---

- [ClassLoader](#ClassLoader)

## <span id="ClassLoadere">ClassLoader<span id="包-package">

Java 默认提供的三个 ClassLoader

1. BootStrap ClassLoader

称为启动类加载器，是 Java 类加载层次中最顶层的类加载器，负责加载 JDK 中的核心类库，如：rt.jar、resources.jar、charsets.jar 等.

2. Extension ClassLoader：称为扩展类加载器，负责加载 Java 的扩展类库，默认加载 JAVA_HOME/jre/lib/ext / 目下的所有 jar。

3. App ClassLoader：称为系统类加载器，负责加载应用程序 classpath 目录下的所有 jar 和 class 文件。

注意： 除了 Java 默认提供的三个 ClassLoader 之外，用户还可以根据需要定义自已的 ClassLoader，而这些自定义的 ClassLoader 都必须继承自 java.lang.ClassLoader 类，也包括 Java 提供的另外二个 ClassLoader（Extension ClassLoader 和 App ClassLoader）在内，但是 Bootstrap ClassLoader 不继承自 ClassLoader，因为它不是一个普通的 Java 类，底层由 C++ 编写，已嵌入到了 JVM 内核当中，当 JVM 启动后，Bootstrap ClassLoader 也随着启动，负责加载完核心类库后，并构造 Extension ClassLoader 和 App ClassLoader 类加载器。

* [深入分析 Java ClassLoader 原理](http://www.importnew.com/15362.html)