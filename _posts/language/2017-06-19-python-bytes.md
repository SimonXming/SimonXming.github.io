---
layout: post
title: python 二进制
category: 语言
tags: python 二进制
keywords: bytes
description:
---

显然，在 python2 和 python3 中，二进制相关内容并不完全一样。

# Python2

python2 本身并没有对二进制进行支持，不过提供了一个模块来弥补，就是 struct 模块。

```python
>>> type(b"abc")
<type 'str'>
>>> str == bytes
True
```

python 没有二进制类型，但可以存储二进制类型的数据，就是用 string 字符串类型来存储二进制数据，这也没关系，因为 string 是以 1 个字节为单位的。

```python
import struct
a = 12.34
# 将 a 变为二进制
bytes = struct.pack('i',a)
```

此时 bytes 就是一个 string 字符串，字符串按字节同 a 的二进制存储内容相同。再进行反操作。

现有二进制数据 bytes，（其实就是字符串），将它反过来转换成 python 的数据类型：

```python
a,=struct.unpack('i',bytes) # 注意，unpack 返回的是 tuple
```

```python
a = 'hello'
b = 'world!'
c = 2
d = 45.123
bytes = struct.pack('5s6sif',a,b,c,d)
```
此时的 bytes 就是二进制形式的数据了，可以直接写入文件比如 `binfile.write(bytes)`
然后，当我们需要时可以再读出来，`bytes=binfile.read()`
再通过 `struct.unpack()` 解码成 python 变量

```python
a, b, c, d = struct.unpack('5s6sif', bytes)
```

'5s6sif'这个叫做 fmt，就是格式化字符串，由数字加字符构成，5s 表示占 5 个字符的字符串，2i，表示 2 个整数等等，下面是可用的字符及类型，ctype 表示可以与 python 中的类型一一对应。

Format|C Type|Python|字节数
------|----|-----|----
x|pad byte|no value|1
c|char|string of length 1|1
b|signed char|integer|1
B|unsigned char|integer|1
?|_Bool|bool|1
h|short|integer|2
H|unsigned short|integer|2
i|int|integer|4
I|unsigned int|integer or long|4
l|long|integer|4
L|unsigned long|long|4
q|long long	|long|8
Q|unsigned long long|long|8
f|float|float|4
d|double|float|8
s|char[]|string|1
p|char[]|string|1
P|void *|long|

最后一个可以用来表示指针类型的，占 4 个字节

为了同 c 中的结构体交换数据，还要考虑有的 c 或 c++ 编译器使用了字节对齐，通常是以 4 个字节为单位的 32 位系统，故而还提供了

Character|Byte order|Size and alignment
--------|-------|------------------
@|native|native            凑够 4 个字节
=|native|standard        按原字节数
<|little-endian|standard        按原字节数
>|big-endian|standard       按原字节数
!|network (= big-endian)|standard       按原字节数
使用方法是放在 fmt 的第一个位置，就像'@5s6sif'

我们使用处理二进制文件时，需要用如下方法
```python
binfile=open(filepath,'rb')    # 读二进制文件
或
binfile=open(filepath,'wb')    写二进制文件
```
那么和 binfile=open(filepath,'r') 的结果到底有何不同呢？

不同之处有两个地方：

1. 第一，使用'r'的时候如果碰到'0x1A'，就会视为文件结束，这就是 EOF。使用'rb'则不存在这个问题。即，如果你用二进制写入再用文本读出的话，如果其中存在'0X1A'，就只会读出文件的一部分。使用'rb'的时候会一直读到文件末尾。
2. 第二，对于字符串 x='abc\ndef'，我们可用 len(x) 得到它的长度为 7，\n 我们称之为换行符，实际上是'0X0A'。当我们用'w'即文本方式写的时候，在 windows 平台上会自动将'0X0A'变成两个字符'0X0D'，'0X0A'，即文件长度实际上变成 8.。当用'r'文本方式读取时，又自动的转换成原来的换行符。如果换成'wb'二进制方式来写的话，则会保持一个字符不变，读取时也是原样读取。所以如果用文本方式写入，用二进制方式读取的话，就要考虑这多出的一个字节了。'0X0D'又称回车符。

linux 下不会变。因为 linux 只使用'0X0A'来表示换行。

# Python3

Bytes 对象是由单个字节作为基本元素（8 位，取值范围 0-255）组成的序列，为不可变对象。

Bytes 对象只负责以二进制字节序列的形式记录所需记录的对象，至于该对象到底表示什么（比如到底是什么字符）则由相应的编码格式解码所决定。我们可以通过调用 bytes() 类（没错，它是类，不是函数）生成 bytes 实例，其值形式为 `b'xxxxx'`，其中 'xxxxx' 为一至多个转义的十六进制字符串（单个 x 的形式为：\xHH，其中 \x 为小写的十六进制转义字符，HH 为二位十六进制数）组成的序列，每个十六进制数代表一个字节（八位二进制数，取值范围 0-255），对于同一个字符串如果采用不同的编码方式生成 bytes 对象，就会形成不同的值：

```python
>>> a = "徐"
>>> ord(a)
24464
>>> b = bytes(a, "utf-8")
>>> b
b'\xe5\xbe\x90'
>>> c = bytes(a, "gb2312")
>>> c
b'\xd0\xec'
```

比如上例中的 a 字符串对象，其十进制 unicode 值为 24464，分别使用'utf-8' 和'gb2312' 两种编码格式将其转换成 bytes 对象 b 和 c ，结果 b 和 c 的值是完全不同的，由于基于的编码格式不一致， b c 长度甚至都不相同，前者有 3 个字节长度，后者有 2 个字节长度：

```python
>>> print(len(b), len(c))
3 2
```

另外，对于 ASCII 字符串，可以直接使用 b'xxxx' 赋值创建 bytes 实例，但对于非 ASCII 编码的字符则不能通过这种方式创建 bytes 实例：

```python
>>> d = b"徐"
  File "<stdin>", line 1
SyntaxError: bytes can only contain ASCII literal characters.
>>> d = b"http://google.com"
>>> d
b'http://google.com'
```

由于 bytes 是序列，因此我们可以通过索引或切片访问它的元素：

```python
>>> b[0]
229
>>> b[2]
144
>>> b[0:1]
b'\xe5'
```

可以发现如果以单个索引的形式访问元素，其会直接返回单个字节的十进制整数，而以序列片段的形式访问时，则返回相应的十六进制字符序列。

对于 bytes 实例，如果需要还原成相应的字符串，则需要借助内置的解码函数 decode()，借助相应的编码格式解码为正常字符串对象，如果采用错误的编码格式解码，则有可能发生错误：

```python
>>> b.decode("utf-8")
'徐'
>>> c.decode("gb2312")
'徐'
>>> b.decode("gb2312")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
UnicodeDecodeError: 'gb2312' codec can't decode byte 0x90 in position 2: incomplete multibyte sequence
```