---
layout: post
title: interface 与 nil 的比较
category: 语言
tags: golang interface
keywords: golang
description:
---

[Go语言第一深坑 - interface 与 nil 的比较](http://studygolang.com/articles/10635)


## interface简介

Go语言以简单易上手而著称，它的语法非常简单，熟悉C++，Java的开发者只需要很短的时间就可以掌握Go语言的基本用法。

interface是Go语言里所提供的非常重要的特性。一个interface里可以定义一个或者多个函数，例如系统自带的io.ReadWriter的定义如下所示：

```go
type ReadWriter interface {
    Read(b []byte) (n int, err error)
    Write(b []byte) (n int, err error)
}
```

任何类型只要它提供了Read和Write的绑定函数实现，Go就认为这个类型实现了这个interface（duck-type），而不像Java需要开发者使用implements标明。

然而Go语言的interface在使用过程中却有一个特别坑的特性，当你比较一个interface类型的值是否是nil的时候，这是需要特别注意避免的问题。

## 一次真实的踩坑

这是我们在GoWorld分布式游戏服务器的开发中，碰到的一个实际的bug。由于GoWorld支持多种不同的数据库（包括MongoDB，Redis等）来保存服务端对象，因此GoWorld在上层提供了一个统一的对象存储接口定义，而不同的对象数据库实现只需要实现EntityStorage接口所提供的函数即可。

```go
// EntityStorage defines the interface of entity storage backends
type EntityStorage interface {
    List(typeName string) ([]common.EntityID, error)
    Write(typeName string, entityID common.EntityID, data interface{}) error
    Read(typeName string, entityID common.EntityID) (interface{}, error)
    Exists(typeName string, entityID common.EntityID) (bool, error)
    Close()
    IsEOF(err error) bool
}
```

以一个使用Redis作为对象数据库的实现为例，函数OpenRedis连接Redis数据库并最终返回一个redisEntityStorage对象的指针。

```go
// OpenRedis opens redis as entity storage
func OpenRedis(url string, dbindex int) *redisEntityStorage {
    c, err := redis.DialURL(url)
    if err != nil {
        return nil
    }

    if dbindex >= 0 {
        if _, err := c.Do("SELECT", dbindex); err != nil {
            return nil
        }
    }

    es := &redisEntityStorage{
        c: c,
    }

    return es
}
```

在上层逻辑中，我们使用OpenRedis函数连接Redis数据库，并将返回的 redisEntityStorage 指针赋值个一个 EntityStorage 接口变量，因为 redisEntityStorage 对象实现了 EntityStorage 接口所定义的所有函数。

```go
var storageEngine StorageEngine // 这是一个全局变量
storageEngine = OpenRedis(cfg.Url, dbindex)
if storageEngine != nil {
    // 连接成功
    ...
} else {
    // 连接失败
    ...
}
```

上面的代码看起来都很正常，OpenRedis在连接Redis数据库失败的时候会返回nil，然后调用者将返回值和nil进行比较，来判断是否连接成功。这个就是Go语言少有的几个深坑之一，因为不管OpenRedis函数是否连接Redis成功，都会运行连接成功的逻辑。

## 寻找问题所在

想要理解这个问题，首先需要理解interface{}变量的本质。在Go语言中，一个interface{}类型的变量包含了2个指针，一个指针指向值的类型，另外一个指针指向实际的值。 我们可以用如下的测试代码进行验证。

```go
// InterfaceStructure 定义了一个interface{}的内部结构
type InterfaceStructure struct {
    pt uintptr // 到值类型的指针
    pv uintptr // 到值内容的指针
}

// asInterfaceStructure 将一个interface{}转换为InterfaceStructure
func asInterfaceStructure (i interface{}) InterfaceStructure {
    return *(*InterfaceStructure)(unsafe.Pointer(&i))
}

func TestInterfaceStructure(t *testing.T) {
    var i1, i2 interface{}
    var v1 int = 0x0AAAAAAAAAAAAAAA
    var v2 int = 0x0BBBBBBBBBBBBBBB
    i1 = v1
    i2 = v2
    fmt.Printf("sizeof interface{} = %d\n", unsafe.Sizeof(i1))
    fmt.Printf("i1 %x %+v\n", i1, asInterfaceStructure(i1))
    fmt.Printf("i2 %x %+v\n", i2, asInterfaceStructure(i2))
    var nilInterface interface{}
    fmt.Printf("nil interface = %+v\n", asInterfaceStructure(nilInterface))
}
```

这段代码的输出如下：

```go
sizeof interface{} = 16
i1 aaaaaaaaaaaaaaa {pt:5328736 pv:825741282816}
i2 bbbbbbbbbbbbbbb {pt:5328736 pv:825741282824}
nil interface = {pt:0 pv:0}
```

所以对于一个interface{}类型的nil变量来说，它的两个指针都是0。这是符合Go语言对nil的标准定义的。在Go语言中，nil是零值（Zero Value），而在Java之类的语言里，null实际上是空指针。关于零值和空指针有什么区别，这里就不再展开了。

当我们将一个具体类型的值赋值给一个interface类型的变量的时候，就同时把类型和值都赋值给了interface里的两个指针。如果这个具体类型的值是nil的话，interface变量依然会存储对应的类型指针和值指针。

```go
func TestAssignInterfaceNil(t *testing.T) {
    var p *int = nil
    var i interface{} = p
    fmt.Printf("%v %+v is nil %v\n", i, asInterfaceStructure(i), i == nil)
}
```

输入如下：

```shell
<nil> {pt:5300576 pv:0} is nil false
```

可见，在这种情况下，虽然我们把一个nil值赋值给interface{}，但是实际上interface里依然存了指向类型的指针，所以拿这个interface变量去和nil常量进行比较的话就会返回false。

## 如何解决这个问题

想要避开这个Go语言的坑，我们要做的就是避免将一个有可能为nil的具体类型的值赋值给interface变量。以上述的OpenRedis为例，一种方法是先对OpenRedis返回的结果进行非-nil检查，然后再赋值给interface变量，如下所示。

```go
var storageEngine StorageEngine // 这是一个全局变量
redis := OpenRedis(cfg.Url, dbindex)
if redis != nil {
    // 连接成功
    storageEngine = redis // 确定redis不是nil之后再赋值给interface变量
} else {
    // 连接失败
    ...
}
```

另外一种方法是让OpenRedis函数直接返回EntityStorage接口类型的值，这样就可以把OpenRedis的返回值直接正确赋值给EntityStorage接口变量。

```go
// OpenRedis opens redis as entity storage
func OpenRedis(url string, dbindex int) EntityStorage {
    c, err := redis.DialURL(url)
    if err != nil {
        return nil
    }

    if dbindex >= 0 {
        if _, err := c.Do("SELECT", dbindex); err != nil {
            return nil
        }
    }

    es := &redisEntityStorage{
        c: c,
    }

    return es
}
```