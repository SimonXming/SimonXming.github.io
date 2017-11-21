---
layout: post
title: Java SE support for Docker CPU and memory limits
category: 语言
tags: java docker
keywords: java memory-limit
description:
---

For those who have been using ___Docker’s CPU and memory limits with Java applications___ may have experienced some challenges. In particular CPU limits since the JVM internally and transparently sets the number of GC threads, and JIT compiler threads. These can be explicitly set via command line options, -XX:ParallelGCThreads and -XX:CICompilerCount. For memory limits, the maximum Java heap size can also be set explicitly via JVM command line option, -Xmx.

However, in cases where none of the aforementioned JVM command line options are specified, ___when a Java application using Java SE 8u121 and earlier, when run in a Docker container___, the JVM will use the underlying host configuration of CPUs and memory when transparently determining the number of GC threads, JIT compiler threads and maximum Java heap to use.

As of ___Java SE 8u131, and in JDK 9___, the JVM is ___Docker-aware___ with respect to Docker CPU limits transparently. That means if -XX:ParalllelGCThreads, or -XX:CICompilerCount are not specified as command line options, the JVM will apply the Docker CPU limit as the number of CPUs the JVM sees on the system. The JVM will then adjust the number of GC threads and JIT compiler threads just like it would as if it were running on a bare metal system with number of CPUs set as the Docker CPU limit. If -XX:ParallelGCThreads or -XX:CICompilerCount are specified as JVM command line options, and Docker CPU limit are specified, the JVM will use the -XX:ParallelGCThreads and -XX:CICompilerCount values.

For Docker memory limits, there is a little more work for the transparent setting of a maximum Java heap. To tell the JVM to be aware of Docker memory limits in the absence of setting a maximum Java heap via -Xmx, there are two JVM command line options required, -XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap. The -XX:+UnlockExperimentalVMOptions is required because in a future release transparent identification of Docker memory limits is the goal. When these two JVM command line options are used, and -Xmx is not specified, the JVM will look at the Linux cgroup configuration, which is what Docker containers use for setting memory limits, to transparently size a maximum Java heap size. FYI, Docker containers also use cgroups configuration for CPU limits too.

These two modifications in Java SE 8u131 and JDK 9 are expected to improve the Java running in Docker experience.

