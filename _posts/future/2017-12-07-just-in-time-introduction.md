---
layout: post
title: 即时编译（JIT)
category: 未来
tags: JIT 编译
keywords: JIT
description:
---


# 即时编译（JIT)

即时编译（英语：Just-in-time compilation），又译及时编译、实时编译，动态编译的一种形式，是一种提高程序运行效率的方法。


## 编译过程

```perl
程序源码 => 词法分析 => 单词流 => 词法分析
                                 ||
解释执行 <= 解释器 <= 指令流  <= 抽象语法树 
                                 ||
目标代码 <= 生成器 <= 中间代码 <=  优化器
```

如今，基于物理机、虚拟机等的语言，大多都遵循这种基于现代经典编译原理的思路，在执行前先对程序源码进行词法解析和语法解析处理，把源码转化为抽象语法树。

对于一门具体语言的实现来说，词法和语法分析乃至后面的优化器和目标代码生成器都可以选择独立于执行引擎，形成一个完整意义的编译器去实现，这类代表是 C/C++ 语言。也可以把抽象语法树或指令流之前的步骤实现一个半独立的编译器，这类代表是 Java 语言。又或者可以把这些步骤和执行引擎全部集中在一起实现，如大多数的 JavaScript 执行器。



### Javac 编译

在 Java 中提到 “编译”，自然很容易想到 Javac 编译器将 `*.java` 文件编译成为 `*.class` 文件的过程，这里的 Javac 编译器称为 __前端编译器__，其他的前端编译器还有诸如 Eclipse JDT 中的增量式编译器 ECJ 等。

相对应的还有**后端编译器**，它在程序运行期间将字节码转变成机器码（现在的 Java 程序在运行时基本都是解释执行加编译执行），如 HotSpot 虚拟机自带的 JIT（Just In Time Compiler）编译器（分 Client 端和 Server 端）。

另外，有时候还有可能会碰到静态提前编译器（AOT，Ahead Of Time Compiler）直接把 *.java 文件编译成本地机器代码，如 GCJ、Excelsior JET 等，这类编译器我们应该比较少遇到。

下面简要说下 Javac 编译（**前端编译**）的过程。

####  词法、语法分析

​    词法分析是将源代码的字符流转变为标记（Token）集合。单个字符是程序编写过程中的的最小元素，而标记则是编译过程的最小元素，关键字、变量名、字面量、运算符等都可以成为标记，比如整型标志 int 由三个字符构成，但是它只是一个标记，不可拆分。

​    语法分析是根据 Token 序列来构造抽象语法树的过程。抽象语法树是一种用来描述程序代码语法结构的树形表示方式，语法树的每一个节点都代表着程序代码中的一个语法结构，如 bao、类型、修饰符、运算符等。经过这个步骤后，编译器就基本不会再对源码文件进行操作了，后续的操作都建立在抽象语法树之上。

#### 填充符号表

完成了语法分析和词法分析之后，下一步就是填充符号表的过程。符号表是由一组符号地址和符号信息构成的表格。符号表中所登记的信息在编译的不同阶段都要用到，在语义分析（后面的步骤）中，符号表所登记的内容将用于语义检查和产生中间代码，在目标代码生成阶段，党对符号名进行地址分配时，符号表是地址分配的依据。

#### 语义分析

语法树能表示一个结构正确的源程序的抽象，但无法保证源程序是符合逻辑的。而语义分析的主要任务是读结构上正确的源程序进行上下文有关性质的审查。语义分析过程分为标注检查和数据及控制流分析两个步骤：

- 标注检查步骤检查的内容包括诸如变量使用前是否已被声明、变量和赋值之间的数据类型是否匹配等。
- 数据及控制流分析是对程序上下文逻辑更进一步的验证，它可以检查出诸如程序局部变量在使用前是否有赋值、方法的每条路径是否都有返回值、是否所有的受查异常都被正确处理了等问题。

#### 字节码生成

字节码生成是 Javac 编译过程的最后一个阶段。字节码生成阶段不仅仅是把前面各个步骤所生成的信息转化成字节码写到磁盘中，编译器还进行了少量的代码添加和转换工作。 实例构造器 `<init>（）`方法和类构造器 `< clinit>（）`方法就是在这个阶段添加到语法树之中的（这里的实例构造器并不是指默认的构造函数，而是指我们自己重载的构造函数，如果用户代码中没有提供任何构造函数，那编译器会自动添加一个没有参数、访问权限与当前类一致的默认构造函数，这个工作在填充符号表阶段就已经完成了）。



## JIT in Java

Java 程序最初是仅仅通过解释器解释执行的，即对字节码逐条解释执行，这种方式的执行速度相对会比较慢，尤其当某个方法或代码块运行的特别频繁时，这种方式的执行效率就显得很低。于是后来在虚拟机中引入了 JIT 编译器（即时编译器）。**当虚拟机发现某个方法或代码块运行特别频繁时，就会把这些代码认定为 “Hot Spot Code”（热点代码），为了提高热点代码的执行效率，在运行时，虚拟机将会把这些代码编译成与本地平台相关的机器码，并进行各层次的优化，完成这项任务的正是 JIT 编译器。**

现在主流的商用虚拟机（如 Sun HotSpot、IBM J9）中几乎都同时包含解释器和编译器（三大商用虚拟机之一的 JRockit 是个例外，它内部没有解释器，因此会有启动相应时间长之类的缺点，但它主要是面向服务端的应用，这类应用一般不会重点关注启动时间）。二者各有优势：

* 当程序需要迅速启动和执行时，解释器可以首先发挥作用，省去编译的时间，立即执行；
* 当程序运行后，随着时间的推移，编译器逐渐会返回作用，把越来越多的代码编译成本地代码后，可以获取更高的执行效率。解释执行可以节约内存，而编译执行可以提升效率。

HotSpot 虚拟机中内置了两个 JIT 编译器：Client Complier 和 Server Complier，分别用在客户端和服务端，目前主流的 HotSpot 虚拟机中默认是采用解释器与其中一个编译器直接配合的方式工作。

运行过程中会被即时编译器编译的 “热点代码” 有两类：

- 被多次调用的方法
- 被多次调用的循环体

两种情况，编译器都是以整个方法作为编译对象，这种编译也是虚拟机中标准的编译方式。要知道一段代码或方法是不是热点代码，是不是需要触发即时编译，需要进行 Hot Spot Detection（热点探测）。目前主要的热点 判定方式有以下两种：

- 基于采样的热点探测：采用这种方法的虚拟机会周期性地检查各个线程的栈顶，如果发现某些方法经常出现在栈顶，那这段方法代码就是 “热点代码”。这种探测方法的好处是实现简单高效，还可以很容易地获取方法调用关系，缺点是很难精确地确认一个方法的热度，容易因为受到线程阻塞或别的外界因素的影响而扰乱热点探测。
- 基于计数器的热点探测：采用这种方法的虚拟机会为每个方法，甚至是代码块建立计数器，统计方法的执行次数，如果执行次数超过一定的阀值，就认为它是 “热点方法”。这种统计方法实现复杂一些，需要为每个方法建立并维护计数器，而且不能直接获取到方法的调用关系，但是它的统计结果相对更加精确严谨。

 在 HotSpot 虚拟机中使用的是第二种——基于计数器的热点探测方法，因此它为每个方法准备了两个计数器：方法调用计数器和回边计数器。

* 方法调用计数器用来统计方法调用的次数，在默认设置下，方法调用计数器统计的并不是方法被调用的绝对次数，而是一个相对的执行频率，即一段时间内方法被调用的次数。


* 回边计数器用于统计一个方法中循环体代码执行的次数（准确地说，应该是回边的次数，因为并非所有的循环都是回边），在字节码中遇到控制流向后跳转的指令就称为 “回边”。

在确定虚拟机运行参数的前提下，这两个计数器都有一个确定的阀值，当计数器的值超过了阀值，就会触发 JIT 编译。**触发了 JIT 编译后，在默认设置下，执行引擎并不会同步等待编译请求完成，而是继续进入解释器按照解释方式执行字节码，直到提交的请求被编译器编译完成为止（编译工作在后台线程中进行）。当编译工作完成后，下一次调用该方法或代码时，就会使用已编译的版本。**

由于方法计数器触发即时编译的过程与回边计数器触发即时编译的过程类似，因此这里仅给出方法调用计数器触发即时编译的流程：

![方法计数器触发过程](http://img.blog.csdn.net/20140108202856171)



## JIT in Javascript

JavaScript 则是利用动态类型 (dynamic typing)。 JavaScript 变量没有类型，而所指定对象的类型在第一次执行时（换言之，动态地）就已判定了。每次在 JavaScript 中存取属性 (property)，或是寻求方法等，必须检查对象的类型，并照着进行处理。

JavaScript 的弹性允许在任何时间，在对象上新增或是删除属性和方法等（请参阅附录）。JavaScript 语言非常动态，而业界的一般看法是动态语言比 C++ 或 Java 等静态语言更难加速。

许多 JavaScript 引擎都使用哈希表（hash table）来存取属性和寻找方法等。换言之，每次存取属性或是寻找方法时，就会使用字符串作为寻找对象哈希表的键 (key)。搜寻哈希表是一个连续动作，包含从散列 (hashing) 值中判定数组内位置，然后查看该位置的键值（key）是否符相等。然后可以使用位移直接读取数据的数组比较起来，利用此方法存取较费时。

在 C++ 中，源代码需要经过编译才能执行，在生成本地代码的过程中，变量的地址和类型已经确定，运行本地代码时利用数组和位移就可以存取变量和方法的地址，不需要再进行额外的查找，几个机器指令即可完成，节省了确定类型和地址的时间。JavaScript 和 C++ 有以下几个区别：

- 编译确定位置，C++ 编译阶段确定位置偏移信息，在执行时直接存取，JavaScript 在执行阶段确定，而且执行期间可以修改对象属性；
- 偏移信息共享，C++ 有类型定义，执行时不能动态改变，可共享偏移信息，JavaScript 每个对象都是自描述，属性和位置偏移信息都包含在自身的结构中；
- 偏移信息查找，C++ 查找偏移地址很简单，在编译代码阶段，对使用的某类型成员变量直接设置偏移位置，JavaScript 中使用一个对象，需要通过属性名匹配才能找到相应的值，需要更多的操作。

而使用动态类型的其他语言，还有 Smalltalk 和 Ruby 等。这些语言基本上也是搜寻哈希表，但它们利用类来缩短搜寻时间。然而，JavaScript 没有类。除了「Numbers」指示数字值、「Strings」为字符串以及其他少数几种类型外，其他对象都是「Object」型。程序员无法宣告类型（类），因此无法使用明确的类型来加速处理。

### V8

V8 是一个由 google 德国开发中心开发的 **JavaScript engine**. It is [**open source**](https://code.google.com/p/v8/wiki/Source) and written in **C++**. It is used for both client side (Google Chrome) and server side (node.js) JavaScript applications.

V8 将抽象语法树通过 JIT 技术（like other JavaScript engines like SpiderMonkey, Rhino (Mozilla) ）转换成本地代码，放弃了在字节码阶段可以进行的一些性能优化，但保证了执行速度。在 V8 生成本地代码后，也会通过 Profiler 采集一些信息，来优化本地代码。虽然，少了生成字节码这一阶段的性能优化，但极大减少了转换时间。

#### 工作过程

V8 引擎在执行 JavaScript 的过程中，主要有两个阶段：编译和运行，与 C++ 的执行前完全编译不同的是，JavaScript 需要在用户使用时完成编译和执行。在 V8 中，JavaScript 相关代码并非一下完成编译的，而是在某些代码需要执行时，才会进行编译，这就提高了响应时间，减少了时间开销。

在 V8 引擎中，源代码先被解析器转变为抽象语法树 (AST)，然后使用 JIT 编译器的全代码生成器从 AST 直接生成本地可执行代码。**这个过程不同于 JAVA 先生成字节码或中间表示，减少了 AST 到字节码的转换时间，提高了代码的执行速度。但由于缺少了转换为字节码这一中间过程，也就减少了优化代码的机会。**

由于 V8 缺少了生成中间代码这一环节，缺少了必要的优化，为了提升性能，V8 会在生成本地代码后，使用数据分析器 (profiler) 采集一些信息，然后根据这些数据将本地代码进行优化，生成更高效的本地代码，这是一个逐步改进的过程。同时，当发现优化后代码的性能还不如未优化的代码，V8 将退回原来的代码，也就是优化回滚。

先根据需要编译和生成这些本地代码，也就是使用编译阶段那些类和操作。在 V8 中，函数是一个基本单位，当某个 JavaScript 函数被调用时，V8 会查找该函数是否已经生成本地代码，如果已经生成，则直接调用该函数。否则，V8 引擎会生成属于该函数的本地代码。这就节约了时间，减少了处理那些使用不到的代码的时间。其次，执行编译后的代码为 JavaScript 构建 JS 对象，这需要 Runtime 类来辅组创建对象，并需要从 Heap 类分配内存。再次，借助 Runtime 类中的辅组函数来完成一些功能，如属性访问等。最后，将不用的空间进行标记清除和垃圾回收。

#### 功能模块

* 优化回滚
* 隐藏类与内嵌缓存
* 内存管理与垃圾回收
* 快照

## JIT in Python

#### CPython with JIT - Pyjion

Pyjion is Designing a JIT API for CPython。它目前的项目目标有三个：

1. **Add a C API to CPython for plugging in a JIT** （ [Implement in python3.6](https://python.readthedocs.io/en/latest/whatsnew/3.6.html#pep-523-adding-a-frame-evaluation-api-to-cpython) )


1. Develop a JIT module using CoreCLR utilizing the C API mentioned in goal #1（[代码](https://github.com/Microsoft/Pyjion/tree/master/Pyjion)） <- 概念验证用
2. Develop a C++ framework

> 目标 1 很简单，就是给 CPython 添加一组新的 C API 及其实现，来为外部的 JIT 编译器提供接入 CPython 运行时的钩子。这部分目前设计和实现都很直观，看看上面的代码链接的 patch 就知道它是啥了——在解释器入口处添加钩子，当有 JIT 编译器注册进来时，一个函数在即将开始被解释执行时会先尝试 JIT 编译，如果成功以后就执行 JIT 出来的机器码；如果不成功就会把该函数标记为不可 JIT 编译，以后就不再尝试了。
>
> 目前这 C API 并不太灵活，只允许以 Python 函数为单元来编译，编译必须对整个函数成功，否则就得整个函数留在解释器里跑。这个 API 没有考虑到在函数中间跳进 JIT 编译的代码（On-Stack Replacement，OSR）或从 JIT 编译的代码中途跳回到解释器（deoptimization）之类的需求。
>
> 目标 2 的其实是要使用 CoreCLR 里带着的 [RyuJIT 编译器](https://github.com/dotnet/coreclr/blob/master/Documentation/botr/ryujit-overview.md) 来为 CPython 服务。但是当前的 RyuJIT 的实现依赖了 CLR / CoreCLR 提供的 JIT 编译器接口，所以要单独使用 RyuJIT 的话，得要把原本由 CLR / CoreCLR 提供的一些服务 / 接口给模拟出来才行。这个模拟层在 Pyjion 代码里就是 CExecutionEngine、CorJitInfo 等类。
>
> 换言之，Pyjion 自身在 pyjit.dll 中，而它并不真的需要依赖整个 CoreCLR（主体位于 coreclr.dll），而只需要其中的 RyuJIT（位于 clrjit.dll）及其必须依赖的库（例如 gcinfo），然后提供 CExecutionEngine、CorJitInfo 等类的实现给 RyuJIT 模拟出它所依赖 CoreCLR 的一些功能。
>
> Pyjion 这种实现 JIT 编译的方式，实际的效果是把一个 Python 函数的字节码全部粘合到一起，去掉了解释器循环自身的开销，但是大部分复杂的操作还是调用回到 CPython 运行时去处理的。
>
> 要说在语义层面上的优化，Pyjion 尝试了给 CPython 添加 [tagged pointer](https://github.com/Microsoft/Pyjion/blob/2450d32ddefc14a884e5d76d88df08222ae0dbb6/Pyjion/taggedptr.h) 来减少小整数的内存开销，顺带提高运行性能（因为实际数据就伪装在指针里，离运算更近了）。但为了保证兼容性，tagged pointer 只在被 JIT 编译的函数内部使用，一到 [return_value](https://github.com/Microsoft/Pyjion/blob/2450d32ddefc14a884e5d76d88df08222ae0dbb6/Pyjion/absint.cpp%23L3051)之类的要暴露（escape）出去的地方就还是装箱（box）回到原本的对象形态。对应的 intrinsic 实现在 [TAGGED_METHOD 宏](https://github.com/Microsoft/Pyjion/blob/2450d32ddefc14a884e5d76d88df08222ae0dbb6/Pyjion/intrins.cpp%23L2291)里（例如 [PyJit_Add_Int()](https://github.com/Microsoft/Pyjion/blob/2450d32ddefc14a884e5d76d88df08222ae0dbb6/Pyjion/pycomp.cpp%23L1277)就是这样来的）。
>
> 原本 CPython 解释器在解释执行每 N 条字节码指令后都会做些周期性检查，例如是否应该释放 GIL 来给别的线程机会执行。Pyjion 把 Python 代码 JIT 编译后，这些周期性检查就安放在用户代码里的循环回跳（backedge）的地方。这跟 HotSpot VM 的 JIT 编译代码选择的放置 safepoint polling 的位置一样。
>
> 总体来说，Pyjion 采用了一种非常保守的实现方式，很容易保证正确性，但能带来的性能提升也会非常有限。保守是否就意味着容易被接受呢？难说… 搞不好会给人太多想像空间结果很失望。
>
> 在 JIT 编译之外，Pyjion 还有没有向 CPython 注入任何其它东西呢？一点也没有。GIL、GC、监控之类的额外功能一概没碰。

> ### Will this ever ship with CPython?
>
> Goal #1 is explicitly to add a C API to CPython to support JIT compilers. There is no expectation, though, to ship a JIT compiler with CPython. This is because CPython compiles with nothing more than a C89 compiler, which allows it to run on many platforms. But adding a JIT compiler to CPython would immediately limit it to only the platforms that the JIT supports.



#### Resource

* [PEP 523 -- Adding a frame evaluation API to CPython](https://www.python.org/dev/peps/pep-0523/)
* [Lua is register-based VM (single-pass compiler) ](https://techsingular.org/2016/06/10/lua-%E7%9A%84%E8%AF%AD%E8%A8%80%E5%AE%9E%E7%8E%B0%E9%9A%BE%E5%BA%A6/)


* [coconut: An experimental method JIT for CPython 3](https://github.com/davidmalcolm/coconut)
* [CPython / 微软 Pyjion / IBM Python+OMR](https://zhuanlan.zhihu.com/p/20581695)
* [为什么 V8 JavaScript 引擎这么快 ?](http://www.xuanfengge.com/why-v8-so-fast.html)