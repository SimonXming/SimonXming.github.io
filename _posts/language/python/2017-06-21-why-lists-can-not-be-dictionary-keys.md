---
layout: post
title: Why Lists Can't Be Dictionary Keys
category: 语言
tags: python
keywords: python
description:
---

python 的新人们总是想知道为什么，python 同时包括了 tuple 和 list 类型。tuples 可以被作为 dict 的 key，而 lists 却不可以。这是一个有意的设计，了解 python dict 如何工作后才能更好的解释它。

## How Dictionaries Work

Dictionaries, in Python, are also known as "mappings", because they "map" or "associate" key objects to value objects:

```python
# retrieve the value for a particular key
value = d[key]
```

因此，python mappings 必须可以通过给定一个 key object，确定关联这个给定 key 的 value object 是哪个。一个简单的示例，有一个 list 中保存了一系列的 (key, value) 对，在这个 list 中寻找给定值。然而，这个方案在大量 (key, value) 对的情况下是很慢的。从复杂度的角度来说，这个算法的复杂度是 O(n)，n 是这个 list 中 (key, value) 对的数量。

python 的 dict 实现减少了查找算法的复杂度到 O(1)，这种实现依赖于把 key object 作为参数传给一个 hash 函数。这个 hash 函数读取 key object 的内容，并产生一个整数的 hash 值。这个 hash 值用来判定对应哪个 (key, value) 对。伪代码如下：

```python
def lookup(d, key):
    '''dictionary lookup is done in three steps:
       1. A hash value of the key is computed using a hash function.

       2. The hash value addresses a location in d.data which is
          supposed to be an array of "buckets" or "collision lists"
          which contain the (key,value) pairs.

       3. The collision list addressed by the hash value is searched
          sequentially until a pair is found with pair[0] == key. The
          return value of the lookup is then pair[1].
    '''
    h = hash(key)                  # step 1
    cl = d.data[h]                 # step 2
    for pair in cl:                # step 3
        if key == pair[0]:
            return pair[1]
    else:
        raise KeyError, "Key %s not found." % key
```
为了使上述查找算法可以顺利工作，hash 函数必须保证不同的 key 算出的 hash 值是不同的。

```python
for all i1, i2, if hash(i1) != hash(i2), then i1 != i2
```

否则，根据 key object 算出的 hash 值就可能会查找到错误的关联 value。
为了让上面的查找算法工作的更有效率，大多数 key 的 hash 值最好是个小整数（最好是一位）。

## Types Usable as Dictionary Keys

上述讨论应该可以解释为什么 python 需要:

为了可以被用做 dictionary 的 key，key object 必须支持 hash 函数(e.g. through `__hash__`)，等式判断(e.g. through `__eq__` or `__cmp__`)，而且必须满足上述正确性条件。

### Lists as Dictionary Keys

这也就是说，关于 list 为什么不可以被作为 dict 的 key 的简单答案就是，`__hash__` 函数不支持 list object。但是为什么呢？

想象一下什么样子的 hash 函数可以支持 list object。

如果，list 可以通过内存 id id(list) 的值进行 hash 运算，根据 python 的定义这可能是合理有效的 -- 不同 list object 的 hash 值对应不同的 id 值。但是，list 是一种容器，list 的大多数运算也是依赖于 list 的这种特性。因此，通过 list 的 id 来进行 hash 运算会导致下列错误行为：
* 相同内容的 list hash 值不同.
* 在 dict 查找中使用 list 语法是没有意义的，因为会导致一个 KeyError.

如果 list 是通过它们的内容进行 hash 运算的 (as tuples are), 根据 python 的定义这也可能是合理有效的 -- 不同 list object 的 hash 值对应不同的内容。因此，问题不是在于 hash 函数如何定义的，而是在于 list 作为 dict key 时发生了什么，list 被修改了嘛？如果对 list object 的修改导致 hash value 变化（因为改变了 list 的内容）。那么就会出如下问题：

```python
>>> l = [1, 2]
>>> d = {}
>>> d[l] = 42
>>> l.append(3)
>>> d[l]
Traceback (most recent call last):
  File "<interactive input>", line 1, in ?
KeyError: [1, 2, 3]
>>> d[[1, 2]]
Traceback (most recent call last):
  File "<interactive input>", line 1, in ?
KeyError: [1, 2]
```
因此，hashing list 的两种方式都会出现意料之外的副作用，因此 
> The builtin list type should not be used as a dictionary key.

注意，因为 tuple 是不可变对象，它们也就不会有 list 的问题。它们可以通过内容进行 hash 计算，而不用担心修改的问题。因此，tuple 有 `__hash__` 函数，可以作为 dict 的 key。

## User Defined Types as Dictionary Keys

### What about instances of user defined types?

默认情况下，所有用户定义的类型可以作为 dict key，因为 hash(object) 默认指向 id(object), cmp(object1, object2) 默认指向 cmp(id(object1), id(object2))。上面列出关于 list 的情况也是这样，并不能满足要求。为什么用户定义的类型可以以作为 dict key？

1. 在 object 必须放置在 mapping 中的情况下，object 标识通常要比 object 内容重要得多。
2. 在 object 内容真的很重要的情况下，默认设置可以通过重写 `__hash__` `__cmp__` 或 `__eq__` 来实现。

注意，当需要给一个 object 关联一个 value，简单的把 value 设置为 object 的属性是更好的方法。