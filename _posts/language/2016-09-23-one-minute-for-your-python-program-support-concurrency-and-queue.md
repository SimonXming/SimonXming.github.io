---
layout: post
title: 一分钟让你的python程序支持并发和队列
category: 语言
tags: 并发 队列
keywords: 并发
description:
---

工作了的开发同学想必都会给运营、产品等同事跑过数据。在豆瓣，基本每个工程师都在用DPark，原理就是把任务拆分，利用DPark集群，在多台服务器上同时运行这些任务可以快速的获得结果。但是有些需求不能使用DPark，比如有频繁的数据库操作，想象一下，一跑起来就会出现大量集群的进程连接数据库，让数据库压力骤增，甚至影响现有服务；有些需求用DPark有点杀鸡用了宰牛刀的感觉，占用了DPark集群资源，但是不用的话，跑一次任务就得几十分钟；如果酱厂外的同学也想这么爽，或者我们在没有DPark环境的地方（如本地）跑，怎么达到这个目的呢？
有点常识的同学马上说，你这机器上是多核CPU么？是的话，多进程呗。对，确实这样会有非常明显的速度的提高，但是你充分利用了多进程的并发了么？

为啥这么说呢？假设你有1w个小任务，要在一个16核CPU的服务器运行，把任务按某种条件hash之后取16的余数分配给对应的进程，这是不是效率可以提升到单进程的16倍呢？大部分场景下达不到，因为任务不均衡，也就是任务完成的时间是最后一个完成其全部任务的进程结束决定的，也就是那个短板，在它没有完成前，剩下的那15个进程都是闲着，看着它，那这段时间就是没有充分利用。

更有经验的人说，可以使用队列啊。这也是对的，我们把这1w个任务都丢给这个队列，这16个进程执行完就从队列中取，这样就充分利用了吧？确实。我们再深入一下，有些任务是相对独立的，比如写文件，1w个任务写1w个文件，大家互不相关，上述解法够用。但是很多时候执行完任务需要反馈（也就是要执行的结果），这就是还需要额外一个队列或者带锁的全局变量之类的东西存储这个中间值，不要笑我，之前还用SQLite之类的数据库存过这种中间结果，等全部完成后再从数据库把这些结果整理合并。

还要考虑程序的通用性，考虑多进程之间如何良好的通信... 无论是新手还是老手都好烦啊。

DuangDuangDuang... 今天「一分钟让你的程序支持队列和并发」

首先本文的原理可见PYMOTW的Implementing MapReduce with multiprocessing，也就是利用multiprocessing.Pool（进程池）的map方法实现纯Python的MapReduce。由于篇幅我把英语注释去掉：

```python
import collections
import itertools
import multiprocessing


class SimpleMapReduce(object):

    def __init__(self, map_func, reduce_func=None, processes=None):
        self.map_func = map_func
        self.reduce_func = reduce_func
        self.pool = multiprocessing.Pool(processes)

    def partition(self, mapped_values):
        partitioned_data = collections.defaultdict(list)
        for key, value in mapped_values:
            partitioned_data[key].append(value)
        return partitioned_data.items()

    def __call__(self, inputs, chunksize=1):
        map_responses = self.pool.map(self.map_func, inputs, chunksize=chunksize)
        if self.reduce_func is None:
            return
        partitioned_data = self.partition(itertools.chain(*map_responses))
        reduced_values = self.pool.map(self.reduce_func, partitioned_data)
        return reduced_values
```

为了帮助新手理解，我解释下其中几个点：

1. processes：源文章叫做num_worker，现在标准库这个参数已经叫做processes了，如果是None，会赋值成CPU的数量。 启动的进程数量要考虑资源竞争，对数据库的访问压力等多方面内容，有时候多了反而变慢了。

2. chunksize：是每次取任务的数量，任务小的话可以一次批量的多取点。这个是经验值。

3. chain表示把可迭代的对象串连起来组成一个新的大的迭代器。

这个SimpleMapReduce需要传递一个map函数和一个reduce函数，事实上就是执行2次self.pool.map，使用者可以忽略队列的细节（但是严重推荐看一下源码的实现），第二次直接返回结果而已。当然这个例子中reduce函数可以不设置，也就是不关心结果。

那怎么用呢？首先你要明确可拆分的单元，比如解析某些目录下的文件内容，那么每个被解析的文件就可以作为一个子任务；想获得10w用户的某些数据，那么每个用户就是一个子任务。注意单元也可以按照业务特点更集中，比如10w用户我们可以按某种规则分组，100人为一个组，也就是一个单元。

最后我们验证下这种方式是不是最好，首先这里有一个应用日志的目录，目录下有多个日志文件：
```shell
➜  du -sh /logs/bran              
5.5G	/logs/bran
```

需求很简单，遍历目录下的文件，找到符合条件的日志条目数量。代码放在Gist上面就不贴出来了。

我们挨个看看运行效果：

1. simple.py。单进程方式，结果如下：

    COST: 249.42918396

    也就是花了249秒！！那设想下，现在有T级别的日志，更复杂的处理，你得等多久？

2. multiprocessing_queue.py。多进程 + 队列的方式，把每个文件作为任务放入队列，启动X个进程去获取任务，最后放X个None到队列中，如果获取的任务是None，表示任务都执行完了，进程就结束。任务的执行结果放另外一个队列，最后获取全部的执行结果，这次没有放None，而是捕捉get方法超时来判断队列中有没有待执行的任务。同时，也测试了单倍和双倍CPU个数（测试使用的服务器为24核）的进程的执行效果的对比：

    CPU个数：COST: 30.0309579372

    CPU个数 * 2：COST: 32.4717597961

2个结论：

1. 速度比单进程只提高了8倍，其中一个原因是这个服务器并不是闲置的，还在完成其他任务。

2. 可见进程更多并没有提高运行效率，这个需要在实际工作中注意。

3. multiprocessing_pool.py，使用上述的SimpleMapReduce的方式，但是有一点需要注意，SimpleMapReduce的map函数的返回值的每一项都是键、符合数的元组（或列表），而之前的解析函数返回的就是符合的结果，所以我们得简单封装一下下：

```python
def map_wrapper(*args, **kwargs):
    matches = file_parser(*args, **kwargs)
    return [(match, 1) for match in matches]
```
    执行的结果如下：

    COST: 26.9822928905

看到了吧，你只是额外写了一个map_wrapper，就实现了multiprocessing_queue.py里面那一堆代码的功能。
