---
layout: post
title: dir() 与 __dict__，__slots__ 的区别
category: 语言
tags: python
keywords: python
description:
---

首先需要知道的是，`dir()` 是 Python 提供的一个 API 函数，`dir()` 函数会自动寻找一个对象的所有属性，包括搜索 `__dict__` 中列出的属性。

不是所有的对象都有 `__dict__` 属性。例如，如果你在一个类中添加了 `__slots__` 属性，那么这个类的实例将不会拥有 `__dict__` 属性，但是 dir() 仍然可以找到并列出它的实例所有有效属性。

```python
>>> class Foo(object):
...     __slots__ = ('bar', )
...     bar = 'spam'
... 

>>> Foo.__dict__
dict_proxy({'__module__': '__main__', 'bar': 'spam', '__slots__': ('bar',), '__doc__': None})

>>> Foo().__dict__
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'Foo' object has no attribute '__dict__'

>>> dir(Foo)
['__class__', '__delattr__', '__doc__', '__format__', '__getattribute__', '__hash__', '__init__', '__module__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__slots__', '__str__', '__subclasshook__', 'bar']

>>> dir(Foo())
['__class__', '__delattr__', '__doc__', '__format__', '__getattribute__', '__hash__', '__init__', '__module__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__slots__', '__str__', '__subclasshook__', 'bar']
```

同理许多内建类型都没有 `__dict__` 属性，例如 list 没有 `__dict__` 属性，但你仍然可以通过 dir() 列出 list 所有的属性。

```python
>>> [].__dict__
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'list' object has no attribute '__dict__'
>>> dir([])
['__add__', '__class__', '__contains__', '__delattr__', '__delitem__', '__delslice__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__getitem__', '__getslice__', '__gt__', '__hash__', '__iadd__', '__imul__', '__init__', '__iter__', '__le__', '__len__', '__lt__', '__mul__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__reversed__', '__rmul__', '__setattr__', '__setitem__', '__setslice__', '__sizeof__', '__str__', '__subclasshook__', 'append', 'count', 'extend', 'index', 'insert', 'pop', 'remove', 'reverse', 'sort']
```

### dir() 函数和类实例

Python 的实例拥有它们自己的 `__dict__`，而它们对应的类也有自己的 `__dict__`：

```python
>>> class Foo(object):
...     bar = 'spam'
... 
>>> Foo().__dict__
{}
>>> Foo.__dict__
dict_proxy({'__dict__': <attribute '__dict__' of 'Foo' objects>, '__weakref__': <attribute '__weakref__' of 'Foo' objects>, '__module__': '__main__', 'bar': 'spam', '__doc__': None})
```

dir() 函数会遍历 Foo，Foo() 和 object 的 __dict__ 属性，从而为 Foo 类，它的实例以及它所有的被继承类创建一个完整有效的属性列表。

当你对一个类设置属性时，它的实例也会受到影响：

```python
>>> f = Foo()
>>> f.ham
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'Foo' object has no attribute 'ham'
>>> Foo.ham = 'eggs'
>>> f.ham
'eggs'
```

这是因为'ham' 属性被添加到了 Foo 类的 __dict__ 属性中：

```python
>>> Foo.__dict__
dict_proxy({'__module__': '__main__', 'bar': 'spam', 'ham': 'eggs', '__dict__': <attribute '__dict__' of 'Foo' objects>, '__weakref__': <attribute '__weakref__' of 'Foo' objects>, '__doc__': None})
>>> f.__dict__
{}
```

尽管 Foo 的实例 f 自己的 `__dict__` 为空，但它还是能拥有作为类属性的'ham'。Python 中，一个对象的属性查找顺序遵循首先查找实例对象自己，然后是类，接着是类的父类。

所以当你直接对一个实例设置属性时，会看到实例的 `__dict__` 中添加了该属性，但它所属的类的 `__dict__` 却不受影响。

```python
>>> f.stack = 'overflow'
>>> f.__dict__
{'stack': 'overflow'}
>>> Foo.__dict__
dict_proxy({'__module__': '__main__', 'bar': 'spam', 'ham': 'eggs', '__dict__': <attribute '__dict__' of 'Foo' objects>, '__weakref__': <attribute '__weakref__' of 'Foo' objects>, '__doc__': None})
```

dir() 所做的不是查找一个对象的 `__dict__` 属性（这个属性有时甚至都不存在），它使用的是对象的继承关系来反馈一个对象的完整的有效属性。

一个实例的 `__dict__` 属性仅仅是那个实例的局部属性集合，不包含该实例所有有效属性。所以如果你想获取一个对象所有有效属性，你应该使用 dir() 来替代 `__dict__` 或者 `__slots__`。

 

### 关于__slots__

合理使用 `__slots__` 属性可以节省一个对象所消耗的空间。一个普通对象使用一个 dict 来保存它自己的属性，你可以动态地向其中添加或删除属性，但是如果使用 `__slots__` 属性，那么该对象用来保存其自身属性的结构一旦创建则不能再进行任何修改。因此使用 slot 结构的对象节省了一部分开销。虽然有时这是一个很有用的优化方案，但是它也可能没那么有用，因为如果 Python 解释器足够动态，那么它完全可以在向对象添加属性时只请求该对象所使用的 dict。

如何让 CPython 变得足够强大可以自动节省空间而不使用 `__slots__` 属性是一项重大工程，这也许就是为什么它依然存在于 P3k 中的原因吧。

### Python 官方关于 __slots__ 的介绍

在默认情况下，Python 的新类和旧类的实例都有一个字典来存储属性值。这对于那些没什么实例属性的对象来说太浪费空间了，当需要创建大量实例的时候，这个问题变得尤为突出。

因此这种默认做法可以通过在新式类中定义一个 `__slots__` 属性从而得到解决。`__slots__` 声明中包含若干实例变量，并为每个实例预留恰好足够的空间来保存每个变量，因为没有为每个实例都创建一个字典，从而节省空间。

 

### __slots__

 

这个类变量可以是 string，iterable 或者是被实例使用的一连串 string。如果在新式类中定义了 `__slots__`，`__slots__` 会为声明了的变量预留空间，同时阻止该类为它的每个实例自动创建 `__dict__` 和 `__weakref__`。

### 使用 __slots__ 须知

当继承一个没有 `__slots__` 属性的类时，子类会自动拥有 `__dict__` 属性，因此在子类中定义 `__slots__` 是毫无意义的，你可以自由访问子类的 `__dict__` 属性，所有未在 `__slots__` 中声明的属性会保存在 `__dict__` 中。
在缺少 `__dict__` 变量的情况下，实例不能接受新的不在 `__slots__` 声明内的变量作为属性，如果试图给这样的类赋予一个未在 `__slots__` 声明的变量作为属性会抛出 AttributeError。但是如果你确实需要能够动态添加属性，那么将字符串 '`__dict__`' 纳入 `__slots__` 的声明中。如果同时在类中定义 `__dict__` 和 `__slots__` 是不行的，因为 `__slots__` 会阻止 `__dict__` 属性的创建。（本条 Python 2.3 及其以后有效）
在缺少 `__weakref__` 变量的情况下，定义了 `__slots__` 的类不支持对其实例的弱引用。如果需要，请将字符串 `__weakref__` 纳入 `__slots__` 声明中。（本条 Python 2.3 及其以后有效）
`__slots__` 的实现原理是在 class 级别为其所声明的每个变量创建 descriptor，由此带来的结果就是，类属性不能用于为 `__slots__` 声明中的实例变量设置默认值，否则类属性会覆写描述符的赋值功能。
`__slots__` 声明只对它所处的类有效，因此，含有 `__slots__` 的类的子类会自动创建一个 `__dict__`，除非在子类中也声明一个 `__slots__` （在这个 `__slots__` 声明应该只包含父类未声明的变量）。
如果一个类中定义了一个在基类中相同的变量，那么子类实例将不能访问基类中定义的实例变量，除非直接从基类中读取描述符。在将来，可能会添加一个检查来阻止这种情况。
非空的 `__slots__` 对某些类无效，某些类是指源自含有长度属性的内建类型，例如 long, str 和 tuple。
任何非字符串的可迭代的对象都可以赋值给 `__slots__` 。具有映射特性的对象也可以赋值为 `__slots__`。但是，在将来，每个键的值可能会赋予特别的含义。

`__class__` 赋值只有当两个类都具有相同的 `__slots__` 时才有效。（本条 Python 2.6 及其以后有效）