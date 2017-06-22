---
layout: post
title: Some question about python design ?
category: 语言
tags: python 设计
keywords: python
description:
---

# 0x01

### Q-1: why types (str, int, dict, ...) `__dict__` attribute is dict_proxy object in python2 (or mappingproxy object in python3.3+) ?

```python
>>> str.__dict__
dict_proxy({'__add__': <slot wrapper '__add__' of 'str' objects>,
            '__contains__': <slot wrapper '__contains__' of 'str' objects>,
            .....
            'zfill': <method 'zfill' of 'str' objects>})
>>> type(str.__dict__)
<type 'dictproxy'>
>>> s = "abc"
>>> s.__dict__
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'str' object has no attribute '__dict__'
```

#### 问题起源

在 Python 中对于某些 object `__dict__` 属性是只读的，比如对于 type object。然而，在 [Python2.5-2.6](http://bugs.python.org/issue1303614) 之前，还是有一些一般性方法可以获取和改变 `__dict__` 属性的(without hacking with
gc.get_referrents(), that is)。这会导致一些令人费解的错误。

dictproxy 是为了用于保证 `class.__dict__` 的 keys 必须是 strings, proxy 的机制防止了对于 `class.__dict__` 的写入操作, 因此只有 setattr() 可以被用于添加属性, `class.__setattr__` 的实现确保了 keys-must-be-strings 的限制.

如果我们不使用一些 proxy 的机制，那么 `__dict__`，`class.__dict__` 就可以被写入了。如果可以写入，也就可以被删除，而 `class.__dict__` 中的属性被删除可能会导致解释器崩溃。

> The `__dict__` attribute of some objects is read-only,
e.g. for type objects.  However, there is a generic
way to still access and modify it (without hacking with
gc.get_referrents(), that is).  This can lead to
obscure crashes.  Attached is an example that shows
a potential "problem" involving putting strange keys
in the `__dict__` of a type.
>
> This is probably very minor anyway.  If we wanted to
fix this, we would need a `__dict__` descriptor that
looks more cleverly at the object to which it is
applied.
> 
> BTW the first person who understand why the attached
program crashes gets a free coffee.
>
> ------- [Armin Rigo] [Bypassing __dict__ readonlyness](http://bugs.python.org/issue1303614) [Python2.5-2.6]


### Q-2: why built-in class instances don't have `__dict__` attribute ?

Instances of types defined in C don't have a `__dict__` attribute by default.

### Q-3: what is the `__dict__['__dict__']` attribute of a Python class?

```python
>>> class A(object):
        x = "1"
        def __init__(self):
            self.x = "2"

>>> a = A()
>>> a.__dict__
{'x': '2'}
>>> type(a.__dict__)
dict
>>> A.__dict__
dict_proxy({'__dict__': <attribute '__dict__' of 'A' objects>,
            '__doc__': None,
            '__init__': <function __main__.__init__>,
            '__module__': '__main__',
            '__weakref__': <attribute '__weakref__' of 'A' objects>,
            'x': '1'})
>>> type(A.__dict__)
dict_proxy
>>> A.__dict__["__dict__"]
<attribute '__dict__' of 'A' objects>
>>> type(A.__dict__["__dict__"])
getset_descriptor
>>> isinstance(A.__dict__["__dict__"], types.GetSetDescriptorType)
True
>>> A.__dict__["__dict__"].__get__(a, A)
{'x': '2'}
>>> a.__dict__
{'x': '2'}
```

首先，`A.__dict__.__dict__` 和 `A.__dict__['__dict__']` 是不同的，`A.__dict__.__dict__` 不存在，`A.__dict__['__dict__']` 是指 class 的实例拥有的 `__dict__` 属性，它是一个描述器，调用它会返回实例的 `__dict__` 属性。简单来说，因为一个实例的 `__dict__` 属性**不能**(why?)保存在实例的 `__dict__` 中，所以需要通过 class 中的一个 descriptor 来调用。（因为 python 是动态语言嘛，`A.__dict__['__dict__']` 是一个 GetsetDescriptor，所以实例的属性是有能力增加的）

1. 对于 class A 的实例 a ，访问 `a.__dict__` 时的背后是通过 `A.__dict__['__dict__']` 实现的（`vars(A)['__dict__']`）
2. 对于 class A，访问 `A.__dict__` <u>理论上</u> 是通过 `type.__dict__['__dict__']` 实现的（`vars(type)['__dict__']`）

完整解释：

class 和实例访问属性都是通过属性操作符 (class or metaclass's `__getattribute__`) 和 `__dict__` 属性/协议实现的。

对于一般的实例对象，`__dict__` 会返回一个保存包含所有实例属性的独立的 dict 实例对象，对 `__getattribute__` 的调用首先会访问这个 dict，并获取相应的实例属性 (这个调用会在通过描述器协议访问 class 属性之前，也会在调用 `__getattr__` 之前)。class 里定义的 `__dict__` 描述器实现了对这个 dict 的访问。

* `x.name` 的调用会按照以下顺序: `x.__dict__['name']`, `type(x).name.__get__(x, type(x))`, `type(x).name`
* `x.__dict__` 会按照同样顺序，但是很明显会跳过 `x.__dict__['name']` 的访问。

因为 `x.__dict__` 不能保存在 `x.__dict__["__dict__"]` 中，对于 `x.__dict__` 的访问就会用描述器协议实现，`x.__dict__` 的值就会保存在实例中的一个特殊字段里。

对于 class 也会面临相同的情况，虽然 `class.__dict__` 是一个伪装成 dict 的特殊的 proxy 对象，`class.__dict__` 也不允许你对它进行
修改或替换行为。这个特殊的 proxy 对象允许你，获取那些定义在 class 而不是 class 的基类中的的属性。

默认情况下，`vars(cls)` 对于一个空类型，返回的对象包含三个描述器，`__dict__` 用于保存实例中的属性，`__weakref__` 是用于 weakref 模块的内部逻辑，`__doc__` 是用于 class 的 docstring。前两个描述器可能会因为定义了 `__slots__` 而消失，没有 `__dict__` and `__weakref__` 属性，反而会有每一个定义在 `__slots__` 的属性。此时，实例的属性不会保存在 dict 中，访问属性将会通过相应的描述器实现。

refs: [What is the __dict__.__dict__ attribute of a Python class?](https://stackoverflow.com/questions/4877290/what-is-the-dict-dict-attribute-of-a-python-class/4877655#4877655)