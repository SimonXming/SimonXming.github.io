---
layout: post
title: Python Types and Objects
category: 语言
tags: python 设计
keywords: python
description:
---
(转自知乎: [https://www.zhihu.com/question/38791962/answer/78172929](https://www.zhihu.com/question/38791962/answer/78172929))
## 导读

* 在面向对象体系里面，存在两种关系：父子关系，类型实例关系
* object 是父子关系的顶端，所有的数据类型的父类都是它
* type 是类型实例关系的顶端，所有对象都是它的实例的
* object是一个type，object is and instance of type。即Object是type的一个实例。
* type是一种object， type is kind of object。即Type是object的子类。

## 正文

object 和 type的关系很像鸡和蛋的关系，先有object还是先有type没法说，obejct和type是共生的关系，必须同时出现的。

在Python里面，所有的东西都是对象的概念。在面向对象体系里面，存在两种关系：

* 父子关系，即继承关系，表现为子类继承于父类，
* 类型实例关系，表现为某个类型的实例化，

在python里要查看一个实例的类型，使用它的`__class__`属性可以查看，或者使用type()函数查看。

先来看看type和object：

```python
>>> object
<type 'object'>
>>> type
<type 'type'>
```

它们都是type的一个实例，表示它们都是类型对象。

在Python的世界中，object是父子关系的顶端，所有的数据类型的父类都是它；type是类型实例关系的顶端，所有对象都是它的实例的。

它们两个的关系可以这样描述：
* object是一个type，object is and instance of type。即Object是type的一个实例。

```python
>>> object.__class__
<type 'type'>
>>> object.__bases__
# object 无父类，因为它是链条顶端。
()
```

* type是一种object， type is kind of object。即Type是object的子类。

```python
>>> type.__bases__
(<type 'object'>,)
>>> type.__class__
# type的类型是自己
<type 'type'>
```

此时，白板上对象的关系如下图：

![图一](/assets/img/post/20170623/type-object-图1.png)

我们再引入list, dict, tuple 这些内置数据类型来看看：

```python
>>> list.__bases__
(<type 'object'>,)
>>> list.__class__
<type 'type'>
>>> dict.__bases__
(<type 'object'>,)
>>> dict.__class__
<type 'type'>
>>> tuple.__class__
<type 'type'>
>>> tuple.__bases__
(<type 'object'>,) 
```

它们的父类都是object，类型都是type。再实例化一个list看看：

```python
>>> mylist = [1,2,3]
>>> mylist.__class__
<type 'list'>
>>> mylist.__bases__
Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    AttributeError: 'list' object has no attribute '__bases__'
```

实例化的list的类型是`<type 'list'>`, 而没有了父类。
把它们加到白板上去：

![图二](/assets/img/post/20170623/type-object-图2.png)


白板上的虚线表示源是目标的实例，实线表示源是目标的子类。
即，左边的是右边的类型，而上面的是下面的父亲。虚线是跨列产生关系，而实线只能在一列内产生关系。

除了type和object两者外。当我们自己去定个一个类及实例化它的时候，和上面的对象们又是什么关系呢？
试一下：

```python
>>> class C(object):
... pass
...
>>> C.__class__
<type 'type'>
>>> C.__bases__
(<type 'object'>,)
实例化
>>> c = C()
>>> c.__class__
<class '__main__.C'>
>>> c.__bases__
Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    AttributeError: 'C' object has no attribute '__bases__'
```

这个实例化的C类对象也是没有父类的属性的。再更新一下白板：

![图三](/assets/img/post/20170623/type-object-图3.png)

* 白板上的第一列，目前只有type，我们先把这列的东西叫Type。
* 白板上的第二列，它们既是第三列的类型，又是第一列的实例，我们把这列的对象叫TypeObject
* 白板上的第三列，它们是第二列类型的实例，而没有父类（`__bases__`）的，我们把它们叫Instance。

要属于第一列的，必须是type的子类，那么我们只需要继承type来定义类就可以了：

```python
>>> class M(type):
...     pass
...
>>> M.__class__
<type 'type'>
>>> M.__bases__
(<type 'type'>,)
```

嗯嗯，M类的类型和父类都是type。这个时候，我们可以把它归到第一列去。那么，要怎么样实例化M类型呢？实例化后它应该出现在那个列？嗯嗯，好吧，刚才你一不小心创建了一个元类，MetaClass！即类的类。如果你要实例化一个元类，那还是得定义一个类：

```python
>>> class TM(object):
...     __metaclass__ = M    # 这样来指定元类。
...
>>> TM.__class__
<class '__main__.M'>
# 这个类不再是type类型，而是M类型的。
>>> TM.__bases__
(<type 'object'>,)
```

好了，现在TM这个类就是出现在第二列的。再总结一下：
* 第一列，元类列，type是所有元类的父亲。我们可以通过继承type来创建元类。
* 第二列，TypeObject列，也称类列，object是所有类的父亲，大部份我们直接使用的数据类型都存在这个列的。
* 第三列，实例列，实例是对象关系链的末端，不能再被子类化和实例化。

## 总结

如果type和object只保留一个，那么一定是object。

只有object 时，第一列将不复存在，只剩下二三列，第二列表示类型，第三列表示实例，这个和大部分静态语言的类型架构类似，如java。
这样的架构将让python 失去一种很重要的动态特性--动态创建类型。

本来，类(第二列的同学)在Python里面是一个对象(typeobject)，对象是可以在运行时动态修改的，所以我们能在你定义一个类之后去修改他的行为或属性！
拿掉第一列后，第二列变成了纯类型，写成怎样的，运行时行为就怎样。在这一点上，并不比静态语言有优势。
