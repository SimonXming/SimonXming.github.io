---
layout: post
title: Memory Management and Circular References in Python
category: 语言
tags: python circular-references
keywords: memory circular-references
description:
---

    -- [since Python 3.4, circular references are handled much better](http://engineering.hearsaysocial.com/2013/06/16/circular-references-in-python/#comment-2882030670)

    Nice post. Note that starting from Python 3.4, circular references are handled much better (docs imply it should be
    rare that they are not collected -- but don't give specifics about how to make that happen). For example the
    example you give is no longer a problem in Python 3.5 (probably not in 3.4 either, but can't test it right now).


## 前言

用像 Python, Ruby 这样的解释型语言编程很方便的一个方面就是，通常情况下，你可以避免处理内存管理相关的事情。然而，有一个众所周知的情况 Python 一定会有内存泄漏，这就是当你在对象创建中声明了一个循环引用，而且在类声明中实现了一个自定义的 \_\_del__ 解构方法。例如，考虑如下例子：

One of the more convenient aspects of writing code in interpreted languages such as Python or Ruby is that you normally can avoid dealing with memory management. However, one known case where Python will definitely leak memory is when you declare circular references in your object declarations and implement a custom __del__ destructor method in one these classes. For instance, consider the following example:

```python
class A(object):
    def __init__(self, b_instance):
      self.b = b_instance

class B(object):
    def __init__(self):
        self.a = A(self)
    def __del__(self):
        print "die"

def test():
    b = B()

test()
```
当函数 test() 被调用时，它声明了一个对象 B，在 B 的 \_\_init__ 函数中，把自己当成变量传给了 A，A 然后在\_\_init__ 函数中声明了对 B 的引用，这就造成了一个循环引用。一般情况下，python 的垃圾收集器会被用于检测上面这样的循环引用，并删除掉它们。然而，因为自定义的 \_\__del__ 方法，垃圾收集器会把这个循环引用相关对象标记为 “uncollectable”。从设计上说，垃圾收集器并不知道循环引用对象的销毁顺序，所以也就不会去处理它们。你可以通过强制垃圾收集器运行，并检查 gc.garbage 列表里有什么来验证上述结论。

When the function test() is invoked, it declares an instance of B, which passes itself to A, which then sets a reference to B, resulting in a circular reference. Normally Python's garbage collector, which is used to detect these types of cyclic references, would remove it. However, because of the custom destructor (the __del__ method), it marks this item as "uncollectable". By design, it doesn't know the order in which to destroy the objects, so leaves them alone (see Python's garbage collection documentation for more background). You can verify this aspect by forcing the Python garbage collector to run and inspecting what is set inside the gc.garbage array:

```python
import gc
gc.collect()
print gc.garbage
[<__main__.B object at 0x7f59f57c98d0>]
```
你可以通过 objgraph 库可视化这些循环引用。

You can also see these circular references visually by using the objgraph library, which relies on Python's gc module to inspect the references to your Python objects. Note that objgraph library also deliberately plots the the custom \_\_del__ methods in a red circle to spotlight a possible issue.

为了避免循环引用，你通常需要使用 weak reference，向 python 解释器声明：如果剩余的引用属于 weak reference，或者使用了 context manager 或 with 语法，那么内存可以被垃圾收集器回收并用于重新声明对象。

To avoid circular references, you usually need to use [weak references](http://docs.python.org/2/library/weakref.html), declaring to the interpreter that the memory can be reclaimed for an object if the remaining references are of these types, or to use context managers and the with statement (for an example of this latter approach, see how it was solved for the happybase library).


## find_circular_references.py

```python
# -*- encoding: utf-8 -*-
from __future__ import print_function

import gc
import traceback
import types
from tornado import web, ioloop, gen
from tornado.http1connection import HTTP1ServerConnection


def find_circular_references(garbage=None):
    """
    从 garbage 中寻找循环引用
    """
    def inner(level):
        """
        处理内层的数据
        """
        for item in level:
            item_id = id(item)
            if item_id not in garbage_ids:
                continue
            if item_id in visited_ids:
                continue
            if item_id in stack_ids:
                candidate = stack[stack.index(item):]
                candidate.append(item)
                found.append(candidate)
                continue

            stack.append(item)
            stack_ids.add(item_id)
            inner(gc.get_referents(item))
            stack.pop()
            stack_ids.remove(item_id)
            visited_ids.add(item_id)

    ######### 开始初始化 ########

    # 获取传入的 garbage 或者通过 gc 模块获取 garbage 列表
    garbage = garbage or gc.garbage

    # 已经找到的循环引用列表 type: list of list
    found = []

    # 存放 item 的堆
    stack = []

    # 存放 item_id 的 set
    stack_ids = set()

    # 保存 garbage 里每个对象的 id
    garbage_ids = set(map(id, garbage))

    # 保存 visited item 的 id
    visited_ids = set()

    ######## 初始化结束 ########

    # 进入递归函数 inner
    inner(garbage)
    inner = None
    return found


class CollectHandler(web.RequestHandler):
    @gen.coroutine
    def get(self):
        # collect_result = None
        collect_result = gc.collect()
        garbage = gc.garbage
        # for i in garbage[:5]:
        #     print(gc.get_referents(i), "\r\n")
        self.write("Collected: {}\n".format(collect_result))
        self.write("Garbage: {}\n".format(len(gc.garbage)))
        for circular in find_circular_references():
            print('\n==========\n Circular \n==========')
            for item in circular:
                print('    ', repr(item))
            for item in circular:
                if isinstance(item, types.FrameType):
                    print('\nLocals:', item.f_locals)
                    print('\nTraceback:', repr(item))
                    traceback.print_stack(item)

class DummyHandler(web.RequestHandler):
    @gen.coroutine
    def get(self):
        self.write('ok\n')
        self.finish()

application = web.Application([
    (r'/dummy/', DummyHandler),
    (r'/collect/', CollectHandler),
], debug=True)

if __name__ == "__main__":
    gc.disable()
    gc.collect()
    gc.set_debug(gc.DEBUG_STATS | gc.DEBUG_LEAK)
    print('GC disabled')

    print("Start on 8888")
    application.listen(8888)
    ioloop.IOLoop.current().start()
```