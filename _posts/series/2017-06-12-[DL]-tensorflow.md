---
layout: post
title: tensorflow 入门
category: 系列
tags: 深度学习 tensorflow
keywords: 深度学习
description:
---

### Intro

某天，当我打开 `jupyter notebook` 时，发现日志出现了下面一些 `warning`:

```shell
...
2017-06-12 13:07:37.222743: W tensorflow/core/platform/cpu_feature_guard.cc:45] The TensorFlow library wasn't compiled to use SSE4.1 instructions, but these are available on your machine and could speed up CPU computations.
2017-06-12 13:07:37.222778: W tensorflow/core/platform/cpu_feature_guard.cc:45] The TensorFlow library wasn't compiled to use SSE4.2 instructions, but these are available on your machine and could speed up CPU computations.
2017-06-12 13:07:37.222784: W tensorflow/core/platform/cpu_feature_guard.cc:45] The TensorFlow library wasn't compiled to use AVX instructions, but these are available on your machine and could speed up CPU computations.
2017-06-12 13:07:37.222789: W tensorflow/core/platform/cpu_feature_guard.cc:45] The TensorFlow library wasn't compiled to use AVX2 instructions, but these are available on your machine and could speed up CPU computations.
2017-06-12 13:07:37.222794: W tensorflow/core/platform/cpu_feature_guard.cc:45] The TensorFlow library wasn't compiled to use FMA instructions, but these are available on your machine and could speed up CPU computations.
...
```

简单理解了一下，好像是有些 cpu 指令集可用于加速计算，但是 tensorflow 并没有在这些指令集上编译，所以用不到这些指令集特性。（:O～ GPU 买不起不用 GPU 加速就算了，CPU 加速要玩啊）

上面那些指令集都代表着什么意思呢？

指令集名称|全名
--------|----
SIMD|Single Instruction Multiple Data，即单指令流多数据流
SSE4.1|Streaming SIMD Extensions 4.1
SSE4.2|Streaming SIMD Extensions 4.2
AVX|Advanced Vector Extensions，即高级向量扩展指令集
AVX2|Advanced Vector Extensions 2
FMA|Fused-Multiply-Add，即积和熔加运算

```shell
# 查看 cpu 信息
sysctl -n machdep.cpu.brand_string
> Intel(R) Core(TM) i7-4870HQ CPU @ 2.50GHz
# 查看 cpu 支持的指令集
sysctl -n machdep.cpu.features
> FPU VME DE PSE TSC MSR PAE MCE CX8 APIC SEP MTRR PGE MCA CMOV PAT PSE36 CLFSH DS ACPI MMX FXSR SSE SSE2 SS HTT TM PBE SSE3 PCLMULQDQ DTES64 MON DSCPL VMX SMX EST TM2 SSSE3 FMA CX16 TPR PDCM SSE4.1 SSE4.2 x2APIC MOVBE POPCNT AES PCID XSAVE OSXSAVE SEGLIM64 TSCTMR AVX1.0 RDRAND F16C
```

拓展阅读: [How CPU Work](http://www.hardwaresecrets.com/how-a-cpu-works/)