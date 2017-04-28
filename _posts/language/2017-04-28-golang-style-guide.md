---
layout: post
title: Go 编码风格指南
category: 语言
tags: golang 编码风格
keywords: 编码风格
description:
---


## 编码规范
虽然 Go 自带的 fmt 工具已经解决了大部分排版的问题，但是在命名规范上仍旧有一些要求。另外，Go 的哲学与传统的面向对象的编程语言也有不一致的地方（如 Java），需要进行理解和适应。

之所以要强调编程规范，不仅仅是想要通过一个统一的约定减少代码的理解成本，更是通过引导来帮助大家接受 Go 的特性和风格，而不是简单用 Java 或者 C++ 的思路来写 Go（不然为什么不去写 Java/C++ 呢）

### 命名

#### 包名应为小写单词

不应有下划线或者混合大小写。正确示例 controllers, models, routers, views

#### 全局变量即参数采用驼峰式命名

全局变量：驼峰式，首字母大写（如果不可导出，则首字母小写）
参数传递：驼峰式，首字母小写
局部变量采用下划线形式

例如

```go
func (c *SettingsController) Post() {
	_ftype := c.GetString("formtype")
    ...
}
```

#### 采用全部大写或者全部小写来表示缩写单词

比如对于 url 这个单词，不要使用

`UrlPony`

而要使用

urlPony 或者 URLPony

#### 单个函数的接口名以 er 为后缀

如 Reader, Writer，其具体的实现则去掉 er，如

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

#### 两个函数的接口名综合两个函数名，如

```go
type WriteFlusher interface {
    Write([]byte) (int, error)
    Flush() error
}
```

#### 三个以上函数的接口名类似于结构体名，如

```go
type Car interface {
    Start([]byte) 
    Stop() error
    Recover()
}
```

#### 结构体方法参数名

统一采用单字母 p 而不是 this, me 或者 self，如

```go
type T struct{} 
func (p *T)Get(){}
```

### 流程控制

#### if 接受初始化语句

约定如下方式建立局部变量

```go
if err := file.Chmod(0664); err != nil {
    return err
}
```

#### for 采用短声明建立局部变量

```go
sum := 0
for i := 0; i < 10; i++ {
    sum += i
}
```

#### range 如果只需要第一项（key），就丢弃第二个

```go
for key := range m {
    if key.expired() {
        delete(m, key)
    }
}
```

#### range 如果只需要第二项，则把第一项置为下划线

```go
sum := 0
for _, value := range array {
    sum += value
}
```

#### 一旦有错误发生，马上返回

```go
f, err := os.Open(name)
if err != nil {
    return err
}
d, err := f.Stat()
if err != nil {
    f.Close()
    return err
}
codeUsing(f, d)
```

#### 不要滥用 panic

不要使用 panic 来做正常的错误处理，应当使用 error 和 多个返回值来进行。

#### 错误消息全小写

错误处理中的字符串应该都是小写，这样在诸如 log.Print("Reading %s: %v", filename, err) 不会出现奇怪的大写。

#### 不要忽略错误

如果一个函数的返回值包括 err，那么不要使用 _ 来忽略它，而应该去检查函数是否执行成功，如果不成功则执行对应的错误处理并返回，只有在确实不希望出现的情况下才使用 panic

#### 无论是参数列表还是返回值，最好加上名称，方便理解（尤其是在有同类型的多个参数的时候）

比如 `func (f *Foo) Location() (float64, float64, error)` 就不如

``` go
// Location returns f's latitude and longitude.
// Negative values mean south and west, respectively.
func (f *Foo) Location() (lat, long float64, err error)
```

来得清晰

### 参数传递

* 对于少量数据，不要传递指针
* 对于大量数据的 struct 可以考虑使用指针
* 传入参数是 map，slice，chan 不要传递指针
    * 因为 map，slice，chan 是引用类型，不需要传递指针的指针

### 注释

每个包都应该有一个包注释，位于 package 之前。如果同一个包有多个文件，只需要在一个文件中编写即可，如：

```go
// Package models 包含应用所需的数据结构及对应的方法
package models
每个以大写字母开头（即可以导出）的方法应该有注释，且已该函数名开头。如：

// Get 会响应对应路由转发过来的 get 请求
func (c *CrewController) Get() {
    ...
}
```

## 常见技巧

### 声明切片

使用 var t []string 而不是 t := []string{} 来声明一个切片，这样如果切片从来没有使用的话，就不会对其分配内存

### 延期执行

Go 的 defer 语句用来调度一个函数调用（被延期的函数），使其在执行 defer 的函数即将返回之前才被运行。这是一种不寻常但又很有效的方法，用于处理类似于不管函数通过哪个执行路径返回，资源都必须要被释放的情况。典型的例子是对一个互斥解锁，或者关闭一个文件。

```go
// Contents returns the file's contents as a string.
func Contents(filename string) (string, error) {
    f, err := os.Open(filename)
    if err != nil {
        return "", err
    }
    defer f.Close()  // f.Close will run when we're finished.
    var result []byte
    buf := make([]byte, 100)
    for {
        n, err := f.Read(buf[0:])
        result = append(result, buf[0:n]...) // append is discussed later.
        if err != nil {
            if err == io.EOF {
                break
            }
            return "", err  // f will be closed if we return here.
        }
    }
    return string(result), nil // f will be closed if we return here.
}
```

对类似 Close 这样的函数调用进行延期，有两个好处。首先，其确保了你不会忘记关闭文件，如果你之后修改了函数增加一个新的返回路径，会很容易犯这样的错。其次，这意味着关闭操作紧挨着打开操作，这比将其放在函数结尾更加清晰。

被延期执行的函数，它的参数（包括接收者，如果函数是一个方法）是在 defer 执行的时候被求值的，而不是在调用执行的时候。这样除了不用担心变量随着函数的执行值会改变，这还意味着单个被延期执行的调用点可以延期多个函数执行。这里有一个简单的例子。

```go
for i := 0; i < 5; i++ {
    defer fmt.Printf("%d ", i)
}
```

被延期的函数按照 LIFO 的顺序执行，所以这段代码会导致在函数返回时打印出 4 3 2 1 0。

## 重点理解

* [数据 - new 与 make](http://www.kancloud.cn/kancloud/effective/72207)
* [接口与方法](http://www.kancloud.cn/kancloud/effective/72210)
* [内嵌 Embedding](http://www.kancloud.cn/kancloud/effective/72212)
* [并发 - Goroutines 与 Channels](http://www.kancloud.cn/kancloud/effective/72213)
* [错误处理](http://www.kancloud.cn/kancloud/effective/72214)

## 参考链接

* [Effective Go 中文版](http://www.hellogcc.org/effective_go.html)
* [Effective Go](https://golang.org/doc/effective_go.html)
* [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)
* [Golang 工程管理](http://kiritor.github.io/2015/06/05/GolangPrjManager/)
