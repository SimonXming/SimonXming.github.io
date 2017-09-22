---
layout: post
title: Python的类方法和静态方法
category: 语言
tags: Python 概念
keywords: Python
description:
---

Python类中有两个特殊的修饰符@classmethod和@staticmethod(即类方法和静态方法), 想要理解需要先理解类属性和实例属性的感念

### 类属性和实例属性
看下面的代码

```
>>> class TestProperty(object):
...     class_property = "class property"
...     
...     def __init__(self):
...         self.instance_property = "instance property"
>>>
>>> test_property = TestProperty()
>>> # 实例可以访问实例属性和类属性
>>> print test_property.class_property
class property
>>> print test_property.instance_property
instance property
>>> # 类可以访问类属性
>>> print TestProperty.class_property
class property
>>> # 但不可以访问实例属性
>>> print TestProperty.instance_property
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: type object 'TestProperty' has no attribute 'instance_property'
>>> # 注意下面的报错
>>> del test_property.class_property
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: class_property
```
总结起来就是:

* 实例属性的初始化在__init__构造器方法中进行(当然也可以通过其它方法赋值)
* 类实例可以访问实例属性和类属性
* 通过类本身则能访问类属性,无法访问实例属性
* 类属性与类实例没有任何关系
* 类属性其实就是其它语言中的静态变量(变量前加static)
* 为什么要有类属性, 一句话命名空间

### 类方法和静态方法
```
>>> class TestMethod(object):
...     class_property = "class property"
...     def __init__(self):
...         self.instance_property = "instance property"
...     def instance_method(self, arvg):
...         print "instance_method(%s, %s)" % (self, arvg)
...         print "instance property: ", self.instance_property
...     @classmethod
...     def class_method(cls, arvg):
...         print "class_method(%s, %s)" % (cls, arvg)
...         print "class property: ", cls.class_property
...     @staticmethod
...     def static_method(arvg):
...         print "static_method(%s)" % arvg
...
>>>
>>> test_method = TestMethod()
>>> # 执行普通方法, 打印出实例的内存地址和实例属性
>>> test_method.instance_method("Hello")
instance_method(<__main__.TestMethod object at 0x7fd8fc56eb90>, Hello)
instance property:  instance property
>>> # 执行类方法
>>> # 通过实例访问, 打印出类本身和类属性
>>> test_method.class_method("Hello")
class_method(<class '__main__.TestMethod'>, Hello)
class property:  class property
>>> # 通过类直接访问, 同上
>>> TestMethod.class_method("Hello")
class_method(<class '__main__.TestMethod'>, Hello)
class property:  class property
>>> # 执行静态方法
>>> # 通过实例访问, 打印出静态方法本身
>>> test_method.static_method("Hello")
static_method(Hello)
>>> # 通过类直接访问, 同上
>>> TestMethod.static_method("Hello")
static_method(Hello)
```

可以看出类方法中cls代表的是类本身, 如果将类方法中的访问类属性的cls去掉, 则会报出NameError的错误

```
>>> class TestMethod(object):
...     class_property = "class property"
...     @classmethod
...     def class_method(cls, arvg):
...         print "class property: ", class_property
...
>>> TestMethod.class_method("Hello")
class property:
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 5, in class_method
NameError: global name 'class_property' is not defined
```

* 类方法是为了访问类属性更加方便
* 类方法和静态方法可以通过类和实例来访问,效果是相同的
* 静态方法跟普通函数没有什么区别

可以通过在实例方法中直接通过TestMethod.class_property来访问, 但是这样不方便也不好维护(如果类名称改了,就会出错), 也可以使用self.class_property来访问, 但注意, 实例本身获取的并不应该是类属性即TestMethod.class_property, 只是因为实例中并没有class_property这个变量, 而是通过查找类属性,发现有同名变量,然后打印出来,通过上面无法实例无法删除类属性是可以看出来的.
当然,如果不信,可以看下面的例子

```
>>> class TestMethod(object):
...     class_property = "class property"
...     def __init__(self):
...         pass
...
>>> test_method = TestMethod()
>>> id(test_method.class_property)
140488040794968
>>> id(TestMethod.class_property)
140488040794968
>>> print test_method.class_property
class property
>>> test_method.class_property = test_method.class_property + "!!!"
>>> print test_method.class_property
class property!!!
>>> print TestMethod.class_property
class property
>>> id(test_method.class_property)
140488040795080
>>> id(TestMethod.class_property)
140488040794968
```
