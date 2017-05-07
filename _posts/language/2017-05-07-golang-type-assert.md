---
layout: post
title: Go 接口——类型断言
category: 语言
tags: Go 类型断言
keywords: 类型断言
description:
---


Java 当中有 instanceof 这样的关键字判断类型 Go 当中自然也有相应的方法来判断类型。

写法为

```golang
value, ok := em.(T)
```

如果确保 em 是同类型的时候可以直接使用

```golang
value := em.(T)
```
一般用于 switch 语句中

* em 代表要判断的变量
* T 代表要判断的类型
* value 代表返回的值
* ok 代表是否为该类型

## CASE 一

em 必须为 interface 类型才可以进行类型断言

比如下面这段代码，

```golang
s := "hello world"
if v, ok := s.(string); ok {
    fmt.Println(v)
}
```
运行报错， invalid type assertion: s.(string) (non-interface type string on left)

在这里只要是在声明时或函数传进来的参数不是 interface 类型那么做类型断言都是回报 non-interface 的错误的 所以我们只能通过将 s 作为一个 interface{} 的方法来进行类型断言，如下代码所示：
```golang
x := "hello world"
if v, ok := interface{}(x).(string); ok { // interface{}(x):把 x 的类型转换成 interface{}
    fmt.Println(v)
}
```
将 s 显示的转换为 interface{} 接口类型则可以进行类型断言了

## CASE 二

当函数作为参数并且被调用函数将参数类型指定为 interface{} 的时候是没有办法直接调用该方法的，如下面代码，
```shell
package main

import (
    "fmt"
)

func ServeHTTP(s string) {
    fmt.Println(s)
}

type Handler func(string)

func panduan(in interface{}) {
    Handler(in)("wujunbin")
}

func main() {
    panduan(Handler(ServeHTTP))
}
```
运行报错，
```shell
./assert.go:14: cannot convert in (type interface {}) to type Handler: need type assertion
```
改正如下，
```golang
package main

import (
    "fmt"
)

func ServeHTTP(s string) {
    fmt.Println(s)
}

type Handler func(string) //使用 type 定义一个函数类型

func panduan(in interface{}) {
    v, ok := in.(Handler)
    if ok {
        v("hello world")
    } else {
        panic("assert fail")
    }
}

func main() {
    panduan(Handler(ServeHTTP))
}
```
只有让传进来的 in 参数先与 Handler 进行类型判断如果返回值是 OK 则代表类型相同才能进行对应的方法调用。

## CASE 三

进行类型断言之后如果断言成功，就只能使用该类型的方法。 比如对一个结构体 S 进行与 A 接口断言，S 实际上实现了 A B 两个接口，A interface 具有 a() 方法，B interface 具有 b() 方法，如果结构体 S 作为参数被传入一个函数中并且在该函数中是 interface{} 类型，那么进行与 A 的类型断言之后就只能调用 a() 而不能调用 b()，因为编译器只知道你目前是 A 类型却不知道你目前也是 B 类型。看下面这个例子，

```golang
package main

import (
    "fmt"
)

type InterfaceA interface {
    AA()
}

type InterfaceB interface {
    BB()
}

// Won 实现了接口 AA 和 BB
type Won struct{}

func (w Won) AA() {
    fmt.Println("AA")
}

func (w Won) BB() {
    fmt.Println("BB")
}

func info(in interface{}) {

    //必须经过 类型断言，才能调用相应的方法
    v, ok := in.(InterfaceA) //必须进行相应的断言才能进行函数的调用，不然编译器不知道是什么具体类型
    if ok {
        v.AA()
    } else {
        fmt.Println("assert fail")
    }
}

func main() {
    var x InterfaceA = Won{}
    info(x)

    var y Won = Won{}
    info(y)
}
```

## CASE 四

switch 与类型断言的结合使用还是比较方便的，如下所示，

```golang
package main

import (
    "fmt"
)

type Element interface{}

func main() {
    var e Element = 100
    switch value := e.(type) {
    case int:
        fmt.Println("int", value)
    case string:
        fmt.Println("string", value)
    default:
        fmt.Println("unknown", value)
    }
}
```
