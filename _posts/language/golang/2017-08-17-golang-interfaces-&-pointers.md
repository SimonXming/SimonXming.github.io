---
layout: post
title: Go interfaces & pointers
category: 语言
tags: golang interface pointer
keywords: golang
description:
---

Interface in Golang is a spec that defines the behavior of a given object.
I came across the http.Handler interface

```go
type Handler interface {
        ServeHTTP(ResponseWriter, *Request)
}
```

我对这个方法签名很好奇 , 为什么接受 Request 为一个 pointer type 但是 ResponseWriter 为一个 value type.

The Request is an actual struct with 指针接收方法, so passing as a pointer to the method makes sense.

为什么 ResponseWriter 不传递一个指针. 在上面 Handler 的这种情况下, ResponseWriter 对象是作为一个值类型传递还是作为一个指针类型传递? 逻辑上讲，对于那些收到的 response 应该作为一个指针变量传递，方便多个 handlers 可以往同一个 response 里写.

> 注意，在 Go 里所有传递的都是值, 即使是指针. 指针就是包含内存地址的值的特殊类型，但是传递指针也是通过传递值实现的.

Further inspection revealed that ResponseWriter 是一个 interface 类型,

```go
ResponseWriter interface {
    Header() Header
    Write([]byte) (int, error)
    WriteHeader(int)
}
```

在运行时，一个 interface 持有一个实现了这个接口的对象。具体到代码中，它持有一个两行的表格，一个指向当前 interface 相关的类型的指针，一个关联具体数据的指针。

一个具体的实现和 interface 之间没有直接的绑定关系。go 可以声明一个 interface 的实现并且提供编译时类型检查。

But it’s still not clear if the writer parameter is a pointer or a value in ServeHTTP method. Lets try out a few examples with interfaces

```go
//(1) The interface
type Mutable interface {
    mutate(newValue string) error
}
//(2) Struct
type Data struct {
    name string
}
//(3) Implements the interface with a pointer receiver
func (d *Data) mutate(newValue string) error {
    d.name = newValue
    return nil
}
//(4) Function that accepts the interface
// 函数定义接受 interface 类型，不管接受的是
func mutator(mute Mutable) error {
    return mute.mutate("mutate")
}
func main() {
    d := Data{name: "fresh"}
    fmt.Println(d.name) //fresh
    //(5) 作为一个 pointer 类型传递
    mutator(&d)
    fmt.Println(d.name) //mutate
}
```

在步骤 3 中，Data 类型的指针实现了 Mutable interface，使得 Data 对象允许了改变。

在步骤 4 中，mutator 函数接受了 Mutable interface 类型的参数，然后进行了实际的改变操作。注意，mutator 函数的参数并不是一个指针类型，而是普通类型（interface 类型）。

然而我们在步骤 5 中给 mutator 函数传递了一个指针。看上去好像 mutator 函数违背了一般的 go 类型检查。举例来说，对于一般类型这种情况就不成立，但是对 interface 类型就成立。

```go
func thisWontWork(s string) { ... }
v := "kirk"
thisWontWork(&v) //compilation error
```

> 一个参数 (a value or pointer) 传递到接受 interface 的 method/function 时，这个参数的类型 (a value or pointer) 应该和参数实现的方法接收方相匹配。

```
           ---------------------------------------------
           | method receiver |  parameter              |
           | ------------------------------------------|
           |    pointer      |  pointer                |
           |-------------------------------------------|
           |    value        |  value                  |
           |                 |  pointer (dereferenced) |
           ---------------------------------------------
```

__value receiver with value param__

```go
//(1) Implements the interface with a value receiver
func (d Data) mutate(newValue string) error {
...
//(2) mutator function is UNCHANGED
func mutator(mute Mutable) error {
//(3) 作为一个 value 类型传递
mutator(d)
fmt.Println(d.name) //fresh
```

__pointer receiver with value param__

```go
//(1) Implements the interface with a pointer receiver
func (d *Data) mutate(newValue string) error {
//(2) Function is UNCHANGED
func mutator(mute Mutable) error {
//(3) 作为一个 value 类型传递. 会产生一个编译时错误.
mutator(d)
```

回到上文中的 Handler，response 结构的 pointer receiver 实现了 ResponseWriter 接口。http server 创建了一个 response object，然后把它的指针传递给了实现了 Handler 接口的 handlers。这允许了 handlers 可以向同一个 response object 中写。

```go
//server.go - showing relevant code
package http
type response struct {
...
//methods of ResponseWriter implemented by response
func (w *response) Header() Header {...
func (w *response) WriteHeader(code int) {...
//readRequest returns a pointer to the response
func ... readRequest(..) (w *response, err error) {
   return &response{...}
}
//serves the request
func (c *conn) serve(ctx context.Context) {
  w_ptr, _ = readRequest(...)
  // w_ptr 是一个 pointer
  ...
  handler.ServeHTTP(w_ptr)
```

因此，看上去尽管 interfaces 技术上总是作为一个 value 传递的，但是一般情况下，interfaces 是作为 pointer 传递的。
这看上对有些人是很显然的，但是对我却不是，特别是当仅仅看 Handler interface 时。它并没有表明任何和期望中参数相关的信息，直到我查看了它的实现。

## 总结

* 接口的实现决定了传递给接收 interface 作为参数的 method/function 时参数的类型 (a pointer or a value).
* 因为两种类型的 receiver 都可以被接受，因此 interface 的实现可以说多种多样的.
* 上面的情况也可能会在类型转变时引起运行时错误。

Some would argue that method receivers should always be pointers but I would counter-argue that the language should provide a way to enforce it in the interface.