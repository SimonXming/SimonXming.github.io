---
layout: post
title: python 源码阅读 - 垃圾回收机制
category: 语言
tags: python gc
keywords: python
description:
---


(转载自：[http://www.wklken.me/](http://www.wklken.me/posts/2015/09/29/python-source-gc.html))
# 概述

无论何种垃圾收集机制, 一般都是两阶段: 垃圾检测和垃圾回收.

在 Python 中, 大多数对象的生命周期都是通过对象的引用计数来管理的.

问题: 但是存在循环引用的问题: a 引用 b, b 引用 a, 导致每一个对象的引用计数都不为 0, 所占用的内存永远不会被回收

要解决循环引用: 必需引入其他垃圾收集技术来打破循环引用. Python 中使用了标记-清除以及分代收集

即, Python 中垃圾回收机制: 引用计数 (主要), 标记清除, 分代收集 (辅助)


# 引用计数

引用计数, 意味着必须在每次分配和释放内存的时候, 加入管理引用计数的动作

引用计数的优点: 最直观最简单, 实时性, 任何内存, 一旦没有指向它的引用, 就会立即被回收

## 计数存储

![PyObject](http://www.wklken.me/imgs/python-source/PyObject.png)

![PyVarObject](http://www.wklken.me/imgs/python-source/PyVarObject.png)

```python
>>> from sys import getrefcount
>>>
>>> a = [1, 2, 3]
>>> getrefcount(a)
2
>>> b = a
>>> getrefcount(a)
3
>>> del b
>>> getrefcount(a)
2
```

## 计数增加

增加对象引用计数, refcnt incr

```c
#define Py_INCREF(op) (                         \
    _Py_INC_REFTOTAL  _Py_REF_DEBUG_COMMA       \
    ((PyObject*)(op))->ob_refcnt++)
```

## 计数减少

减少对象引用计数, refcnt desc
```c

#define _Py_DEC_REFTOTAL        _Py_RefTotal--
#define _Py_REF_DEBUG_COMMA     ,

#define Py_DECREF(op)                                   \
    do {                                                \
        if (_Py_DEC_REFTOTAL  _Py_REF_DEBUG_COMMA       \
        --((PyObject*)(op))->ob_refcnt != 0)            \
            _Py_CHECK_REFCNT(op)                        \
        else                                            \
        _Py_Dealloc((PyObject *)(op));                  \
    } while (0)
```

即, 发现 refcnt 变成 0 的时候, 会调用 `_Py_Dealloc`

```c
PyAPI_FUNC(void) _Py_Dealloc(PyObject *);
#define _Py_REF_DEBUG_COMMA     ,

#define _Py_Dealloc(op) (                               \
    _Py_INC_TPFREES(op) _Py_COUNT_ALLOCS_COMMA          \
    (*Py_TYPE(op)->tp_dealloc)((PyObject *)(op)))
#endif /* !Py_TRACE_REFS */
```

会调用各自类型的 `tp_dealloc`:

1. dict

```c
PyTypeObject PyDict_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "dict",
    sizeof(PyDictObject),
    0,
    (destructor)dict_dealloc,                   /* tp_dealloc */
    ....
}

static void
dict_dealloc(register PyDictObject *mp)
{
    .....
    // 如果满足条件, 放入到缓冲池freelist中
    if (numfree < PyDict_MAXFREELIST && Py_TYPE(mp) == &PyDict_Type)
        free_list[numfree++] = mp;
    // 否则, 调用tp_free
    else
        Py_TYPE(mp)->tp_free((PyObject *)mp);
    Py_TRASHCAN_SAFE_END(mp)
}
```

Python 基本类型的 `tp_dealloc`, 通常都会与各自的缓冲池机制相关, 释放会优先放入缓冲池中 (对应的分配会优先从缓冲池取). 这个内存分配与回收同缓冲池机制相关

当无法放入缓冲池时, 会调用各自类型的 `tp_free`

2. int, 比较特殊
```c
// int, 通用整数对象缓冲池机制
      (freefunc)int_free,                         /* tp_free */
```

3. string
```c
// string
    PyObject_Del,                               /* tp_free */
```

4. dict/tuple/list
```c
    PyObject_GC_Del,                            /* tp_free */
```

5. 自定义对象
```
PyTypeObject PyType_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "type",                                     /* tp_name */
    ...
    PyObject_GC_Del,                            /* tp_free */
};
```
即, 最终, 当计数变为 0, 触发内存回收动作. 涉及函数 `PyObject_Del` 和 `PyObject_GC_Del`, 并且, 自定义类以及容器类型 (dict/list/tuple/set 等) 使用的都是后者 `PyObject_GC_Del`.

## 内存回收 PyObject_Del / PyObject_GC_Del

如果引用计数 = 0:

1. 放入缓冲池
2. 真正销毁, PyObject_Del/PyObject_GC_Del内存操作

这两个操作都是进行内存级别的操作

* PyObject_Del
    > PyObject_Del(op) releases the memory allocated for an object. It does not run a destructor -- it only frees the memory. PyObject_Free is identical.
* PyObject_GC_Del
    ```c
    void
    PyObject_GC_Del(void *op)
    {
        PyGC_Head *g = AS_GC(op);

        // Returns true if a given object is tracked
        if (IS_TRACKED(op))
            // 从跟踪链表中移除
            gc_list_remove(g);
        if (generations[0].count > 0) {
            generations[0].count--;
        }
        PyObject_FREE(g);
    }
    ```
    * IS_TRACKED 涉及到标记 - 清除的机制
    * generations 涉及到了分代回收
    * PyObject_FREE, 则和 Python 底层内存池机制相关

# 标记 - 清除

## 问题: 什么对象可能产生循环引用?

1. 只需要关注关注可能产生循环引用的对象
2. PyIntObject/PyStringObject 等不可能
3. Python 中的循环引用总是发生在 container 对象之间, 所谓 containser 对象即是内部可持有对其他对象的引用: list/dict/class/instance 等等
4. 垃圾收集带来的开销依赖于 container 对象的数量, 必需跟踪所创建的每一个 container 对象, 并将这些对象组织到一个集合中.

## 可收集对象链表

可收集对象链表: 将需要被收集和跟踪的 container, 放到可收集的链表中

任何一个 python 对象都分为两部分: `PyObject_HEAD` + 对象本身数据 (PyObject_HEAD 里实现了双向链表结构)

```c
/* PyObject_HEAD defines the initial segment of every PyObject. */
#define PyObject_HEAD                   \
    _PyObject_HEAD_EXTRA                \
    Py_ssize_t ob_refcnt;               \
    struct _typeobject *ob_type;

//----------------------------------------------------

  #define _PyObject_HEAD_EXTRA            \
      struct _object *_ob_next;           \
      struct _object *_ob_prev;

// 双向链表结构, 垃圾回收
```

可收集对象链表

```c
Modules/gcmodule.c

/* GC information is stored BEFORE the object structure. */
typedef union _gc_head {
    struct {
        // 建立链表需要的前后指针
        union _gc_head *gc_next;
        union _gc_head *gc_prev;
        // 在初始化时会被初始化为 GC_UNTRACED
        Py_ssize_t gc_refs;
    } gc;
    long double dummy;  /* force worst-case alignment */
} PyGC_Head;
```

创建 container 的过程: container 对象 = `pyGC_Head` | `PyObject_HEAD` | `Container Object`

```c
PyObject *
_PyObject_GC_New(PyTypeObject *tp)
{
    PyObject *op = _PyObject_GC_Malloc(_PyObject_SIZE(tp));
    if (op != NULL)
        op = PyObject_INIT(op, tp);
    return op;
}

=> _PyObject_GC_Malloc

#define _PyGC_REFS_UNTRACKED                    (-2)
#define GC_UNTRACKED                    _PyGC_REFS_UNTRACKED

PyObject *
_PyObject_GC_Malloc(size_t basicsize)
{
    PyObject *op;
    PyGC_Head *g;
    if (basicsize > PY_SSIZE_T_MAX - sizeof(PyGC_Head))
        return PyErr_NoMemory();

    // 为 对象本身+PyGC_Head申请内存, 注意分配的size
    g = (PyGC_Head *)PyObject_MALLOC(
        sizeof(PyGC_Head) + basicsize);
    if (g == NULL)
        return PyErr_NoMemory();

    // 初始化 GC_UNTRACED
    g->gc.gc_refs = GC_UNTRACKED;
    generations[0].count++; /* number of allocated GC objects */

    // 如果大于阈值, 执行分代回收
    if (generations[0].count > generations[0].threshold &&
        enabled &&
        generations[0].threshold &&
        !collecting &&
        !PyErr_Occurred()) {

        collecting = 1;
        collect_generations();
        collecting = 0;
    }
    op = FROM_GC(g);
    return op;
}
```

## PyObject_HEAD and PyGC_HEAD

注意, FROM_GC和AS_GC用于 PyObject_HEAD <=> PyGC_HEAD地址相互转换

```c
// => Modules/gcmodule.c

/* Get an object's GC head */
#define AS_GC(o) ((PyGC_Head *)(o)-1)

/* Get the object given the GC head */
#define FROM_GC(g) ((PyObject *)(((PyGC_Head *)g)+1))

// => objimpl.h

#define _Py_AS_GC(o) ((PyGC_Head *)(o)-1)
```

## 问题: 什么时候将 container 放到这个对象链表中

list
```c
// => listobject.c

PyObject *
PyList_New(Py_ssize_t size)
{
    PyListObject *op;
    // 每当创建一个新 list 时
    // 1. 为对象本身和 GC_head 申请内存
    // 2. 初始化 GC_head.gc_refs -> GC_UNTRACED
    // 3. 更新 generations 追踪计数
    // 4. 根据 generations 阈值, 判断是否执行分代回收
    // *5. collect_generations 会调用 collect
    // *6. collect包含标记 - 清除逻辑
    op = PyObject_GC_New(PyListObject, &PyList_Type);
    _PyObject_GC_TRACK(op);
    return (PyObject *) op;
}

// =>  _PyObject_GC_TRACK

// objimpl.h
// 加入到可收集对象链表中

#define _PyObject_GC_TRACK(o) do { \
    PyGC_Head *g = _Py_AS_GC(o); \
    if (g->gc.gc_refs != _PyGC_REFS_UNTRACKED) \
        Py_FatalError("GC object already tracked"); \
    g->gc.gc_refs = _PyGC_REFS_REACHABLE; \
    g->gc.gc_next = _PyGC_generation0; \
    g->gc.gc_prev = _PyGC_generation0->gc.gc_prev; \
    g->gc.gc_prev->gc.gc_next = g; \
    _PyGC_generation0->gc.gc_prev = g; \
    } while (0);
```

## 问题: 什么时候将 container 从这个对象链表中摘除

```c
// Objects/listobject.c

static void
list_dealloc(PyListObject *op)
{
    Py_ssize_t i;
    PyObject_GC_UnTrack(op);
    .....
}

// => PyObject_GC_UnTrack => _PyObject_GC_UNTRACK

// 对象销毁的时候
#define _PyObject_GC_UNTRACK(o) do { \
    PyGC_Head *g = _Py_AS_GC(o); \
    assert(g->gc.gc_refs != _PyGC_REFS_UNTRACKED); \
    g->gc.gc_refs = _PyGC_REFS_UNTRACKED; \
    g->gc.gc_prev->gc.gc_next = g->gc.gc_next; \
    g->gc.gc_next->gc.gc_prev = g->gc.gc_prev; \
    g->gc.gc_next = NULL; \
    } while (0);
```

## 问题: 如何进行标记 - 清除

现在, 我们得到了一个链表

Python 将自己的垃圾收集限制在这个链表上, 循环引用一定发生在这个链表的一群独享之间.

### 概览
`_PyObject_GC_Malloc` 分配内存时, 发现超过阈值, 此时, 会触发 gc, collect_generations 然后调用 collect, collect 包含标记 - 清除逻辑

```c
gcmodule.c

  /* This is the main function.  Read this to understand how the
   * collection process works. */
  static Py_ssize_t
  collect(int generation)
  {
    // 第1步: 将所有比 当前代 年轻的代中的对象 都放到 当前代 的对象链表中
    /* merge younger generations with one we are currently collecting */
    for (i = 0; i < generation; i++) {
        gc_list_merge(GEN_HEAD(i), GEN_HEAD(generation));
    }

    // 第2步: 遍历对象链表, 将每个对象的 gc.gc_ref 值设置为 ob_refcnt
    update_refs(young);

    // 第3步: 计算有效引用计数
    subtract_refs(young);

    // 第4步: 垃圾标记
    gc_list_init(&unreachable);
    move_unreachable(young, &unreachable);

    // 第5步: 将存活对象放入下一代
      /* Move reachable objects to next generation. */
      if (young != old) {
          if (generation == NUM_GENERATIONS - 2) {
              long_lived_pending += gc_list_size(young);
          }
          gc_list_merge(young, old);
      }
      else {
          /* We only untrack dicts in full collections, to avoid quadratic
             dict build-up. See issue #14775. */
          untrack_dicts(young);
          long_lived_pending = 0;
          long_lived_total = gc_list_size(young);
      }

    // 第6步: 执行回收
      delete_garbage(&unreachable, old);

  }
```

### 详情

暂无

## 分代回收

分代收集: 以空间换时间

思想: 将系统中的所有内存块根据其存货的时间划分为不同的集合, 每个集合就成为一个 "代", 垃圾收集的频率随着 "代" 的存活时间的增大而减小 (活得越长的对象, 就越不可能是垃圾, 就应该减少去收集的频率)

Python 中, 引入了分代收集, 总共三个 "代". Python 中, 一个代就是一个链表, 所有属于同一 "代" 的内存块都链接在同一个链表中

表头数据结构

```c
gcmodule.c
  struct gc_generation {
      PyGC_Head head;
      int threshold; /* collection threshold */  // 阈值
      int count; /* count of allocations or collections of younger
                    generations */    // 实时个数
  };
```

三个代的定义

```c
#define NUM_GENERATIONS 3
#define GEN_HEAD(n) (&generations[n].head)

//  三代都放到这个数组中
/* linked lists of container objects */
static struct gc_generation generations[NUM_GENERATIONS] = {
    /* PyGC_Head,                               threshold,      count */
    {{{GEN_HEAD(0), GEN_HEAD(0), 0}},           700,            0},    //700个container, 超过立即触发垃圾回收机制
    {{{GEN_HEAD(1), GEN_HEAD(1), 0}},           10,             0},    // 10个
    {{{GEN_HEAD(2), GEN_HEAD(2), 0}},           10,             0},    // 10个
};

PyGC_Head *_PyGC_generation0 = GEN_HEAD(0);
```
超过阈值, 触发垃圾回收

```c
PyObject *
_PyObject_GC_Malloc(size_t basicsize)
{
    // 执行分配
    ....
    generations[0].count++; /* number of allocated GC objects */  //增加一个
    if (generations[0].count > generations[0].threshold && // 发现大于预支了
        enabled &&
        generations[0].threshold &&
        !collecting &&
        !PyErr_Occurred())
        {
            collecting = 1;
            collect_generations();  //  执行收集
            collecting = 0;
        }
    op = FROM_GC(g);
    return op;
}

=> collect_generations

static Py_ssize_t
collect_generations(void)
{
    int i;
    Py_ssize_t n = 0;

    /* Find the oldest generation (highest numbered) where the count
    * exceeds the threshold.  Objects in the that generation and
    * generations younger than it will be collected. */

    // 从最老的一代, 开始回收
    for (i = NUM_GENERATIONS-1; i >= 0; i--) {  // 遍历所有generation
        if (generations[i].count > generations[i].threshold) {  // 如果超过了阈值
            /* Avoid quadratic performance degradation in number
                of tracked objects. See comments at the beginning
                of this file, and issue #4074.
            */
            if (i == NUM_GENERATIONS - 1
                && long_lived_pending < long_lived_total / 4)
                continue;
            n = collect(i); // 执行收集
            break;  // notice: break了
        }
    }
    return n;
}
```

## Python 中的 gc 模块

gc 模块, 提供了观察和手动使用 gc 的接口

```python
import gc

gc.set_debug(gc.DEBUG_STATS | gc.DEBUG_LEAK)

gc.collect()
```

注意 `__del__` 给 gc 带来的影响。

