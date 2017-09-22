---
layout: post
title: Go 接口——类型转化
category: 语言
tags: Go 类型转化
keywords: 类型转化
description:
---

main.go 的代码如下：

```golang
package main

import (
    "fmt"
)

type myStruct struct {
    name   bool
    userid int64
}

var structZero myStruct
var intZero int
var int32Zero int32
var int64Zero int64
var uintZero uint
var uint8Zero uint8
var uint32Zero uint32
var uint64Zero uint64
var byteZero byte
var boolZero bool
var float32Zero float32
var float64Zero float64
var stringZero string
var funcZero func(int) int
var byteArrayZero [5]byte
var boolArrayZero [5]bool
var byteSliceZero []byte
var boolSliceZero []bool
var mapZero map[string]bool
var interfaceZero interface{}
var chanZero chan int
var pointerZero *int

func main() {
    fmt.Println("structZero: ", structZero)
    fmt.Println("intZero: ", intZero)
    fmt.Println("int32Zero: ", int32Zero)
    fmt.Println("int64Zero: ", int64Zero)
    fmt.Println("uintZero: ", uintZero)
    fmt.Println("uint8Zero: ", uint8Zero)
    fmt.Println("uint32Zero: ", uint32Zero)
    fmt.Println("uint64Zero: ", uint64Zero)
    fmt.Println("byteZero: ", byteZero)
    fmt.Println("boolZero: ", boolZero)
    fmt.Println("float32Zero: ", float32Zero)
    fmt.Println("float64Zero: ", float64Zero)
    fmt.Println("stringZero: ", stringZero)
    fmt.Println("funcZero: ", funcZero)
    fmt.Println("funcZero == nil?", funcZero == nil)
    fmt.Println("byteArrayZero: ", byteArrayZero)
    fmt.Println("boolArrayZero: ", boolArrayZero)
    fmt.Println("byteSliceZero: ", byteSliceZero)
    fmt.Println("byteSliceZero's len?", len(byteSliceZero))
    fmt.Println("byteSliceZero's cap?", cap(byteSliceZero))
    fmt.Println("byteSliceZero == nil?", byteSliceZero == nil)
    fmt.Println("boolSliceZero: ", boolSliceZero)
    fmt.Println("mapZero: ", mapZero)
    fmt.Println("mapZero's len?", len(mapZero))
    fmt.Println("mapZero == nil?", mapZero == nil)
    fmt.Println("interfaceZero: ", interfaceZero)
    fmt.Println("interfaceZero == nil?", interfaceZero == nil)
    fmt.Println("chanZero: ", chanZero)
    fmt.Println("chanZero == nil?", chanZero == nil)
    fmt.Println("pointerZero: ", pointerZero)
    fmt.Println("pointerZero == nil?", pointerZero == nil)
}
```

您可以清楚的了解到各种类型的默认值。如 bool 的默认值是 false，string 的默认值是空串，byte 的默认值是 0，数组的默认就是这个数 组成员类型的默认值所组成的数组等等。然而您或许会发现在上面的例子中：map、interface、pointer、slice、func、chan 的 默认值和 nil 是相等的。关于 nil 可以和什么样的类型做相等比较，您只需要知道 nil 可以赋值给哪些类型变量，那么就可以和哪些类型变量做相等比较。官 方对此有明确的说明：http://pkg.golang.org/pkg/builtin/#Type，也可以看我的另一篇文章：[golang: 详解 interface 和 nil](http://my.oschina.net/goal/blog/194233)。所以现在您应该知道 nil 只能赋值给指针、channel、func、interface、map 或 slice 类型的变量。如果您用 int 类型的变量跟 nil 做相等比较，panic 会找上您。

对于字面量的值，编译器会有一个隐式转换。看下面的例子：

```golang
package main

import (
    "fmt"
)

func main() {
    var myInt int32     = 5
    var myFloat float64 = 0
    fmt.Println(myInt)
    fmt.Println(myFloat)
}
```

对于 myInt 变量，它存储的就是 int32 类型的 5；对于 myFloat 变量，它存储的是 int64 类型的 0。或许您可能会写出这样的代码，但确实不是必须这么做的：
```golang
package main

import (
    "fmt"
)

func main() {
    var myInt int32     = int32(5)
    var myFloat float64 = float64(0)
    fmt.Println(myInt)
    fmt.Println(myFloat)
}
```

在 C 中，大多数类型转换都是可以隐式进行的，比如：

```c++
#include <stdio.h>

int main(int argc, char **argv)
{
        int uid  = 12345;
        long gid = uid;
        printf("uid=%d, gid=%d\n", uid, gid);
        return 0;
}
```

但是在 golang 中，您不能这么做。有个类似的例子：
```golang
package main

import (
    "fmt"
)

func main() {
    var uid int32 = 12345
    var gid int64 = int64(uid)
    fmt.Printf("uid=%d, gid=%d\n", uid, gid)
}
```

很显然，将 uid 赋值给 gid 之前，需要将 uid 强制转换成 int64 类型，否则会 panic。golang 中的类型区分静态类型和底层类型。您可以用 type 关键字定义自己的类型，这样做的好处是可以语义化自己的代码，方便理解和阅读。

```golang
package main

import (
    "fmt"
)

type MyInt32 int32

func main() {
    var uid int32   = 12345
    var gid MyInt32 = MyInt32(uid)
    fmt.Printf("uid=%d, gid=%d\n", uid, gid)
}
```
在上面的代码中，定义了一个新的类型 MyInt32。对于类型 MyInt32 来说，MyInt32 是它的静态类型，int32 是它的底层类型。即使 两个类型的底层类型相同，在相互赋值时还是需要强制类型转换的。可以用 reflect 包中的 Kind 方法来获取相应类型的底层类型。

对于类型转换的截断问题，为了问题的简单化，这里只考虑具有相同底层类型之间的类型转换。小类型 (这里指存储空间) 向大类型转换时，通常都是安全的。下面是一个大类型向小类型转换的示例：

```golang
package main

import (
    "fmt"
)

func main() {
    var gid int32 = 0x12345678
    var uid int8  = int8(gid)
    fmt.Printf("uid=%#x, gid=%#x\n", uid, gid)
}
```
在上面的代码中，gid 为 int32 类型，也即占 4 个字节空间 (在内存中占有 4 个存储单元)，因此这 4 个存储单元的值分别是：0x12, 0x34, 0x56, 0x78。但事实不总是如此，这跟 cpu 架构有关。在内存中的存储方式分为两种：大端序和小端序。大端序的存储方式是高位字节存储在低地址上；小端序的存 储方式是高位字节存储在高地址上。本人的机器是按小端序来存储的，所以 gid 在我的内存上的存储序列是这样的：0x78, 0x56, 0x34, 0x12。如果您的机器是按大端序来存储，则 gid 的存储序列刚好反过来：0x12, 0x34, 0x56, 0x78。对于强制转换后的 uid，肯定是产生了截断行为。因为 uid 只占 1 个字节，转换后的结果必然会丢弃掉多余的 3 个字节。截断的规则是：保留低地址 上的数据，丢弃多余的高地址上的数据。来看下测试结果：

```shell
$ cd $GOPATH/src/typeassert_test
$ go build
$ ./typeassert_test
uid=0x78, gid=0x12345678
```

如果您的输出结果是：
```shell
uid=0x12, gid=0x12345678
```

那么请不要惊讶，因为您的机器是属于大端序存储。

其实很容易根据上面所说的知识来判断是属于大端序或小端序：

```golang
package main

import (
    "fmt"
)

func IsBigEndian() bool {
    var i int32 = 0x12345678
    var b byte  = byte(i)
    if b == 0x12 {
        return true
    }

    return false
}

func main() {
    if IsBigEndian() {
        fmt.Println("大端序")
    } else {
        fmt.Println("小端序")
    }
}
```
```shell
$ cd $GOPATH/src/typeassert_test
$ go build
$ ./typeassert_test
小端序
```
接口的转换遵循以下规则：

* 普通类型向接口类型的转换是隐式的。
* 接口类型向普通类型转换需要类型断言。

普通类型向接口类型转换的例子随处可见，例如：

```golang
package main

import (
    "fmt"
)

func main() {
    var val interface{} = "hello"
    fmt.Println(val)
    val = []byte{'a', 'b', 'c'}
    fmt.Println(val)
}
```

正如您所预料的，"hello" 作为 string 类型存储在 interface{} 类型的变量 val 中，[]byte{'a', 'b', 'c'} 作为 slice 存储在 interface{} 类型的变量 val 中。这个过程是隐式的，是编译期确定的。

接口类型向普通类型转换有两种方式：Comma-ok 断言和 switch 测试。任何实现了接口 I 的类型都可以赋值给这个接口类型变量。由于 interface{} 包含了 0 个方法，所以任何类型都实现了 interface{} 接口，这就是为什么可以将任意类型值赋值给 interface{} 类 型的变量，包括 nil。还有一个要注意的就是接口的实现问题，*T 包含了定义在 T 和 * T 上的所有方法，而 T 只包含定义在 T 上的方法。我们来看一个例子：

```golang
package main

import (
    "fmt"
)

// 演讲者接口
type Speaker interface {
    // 说
    Say(string)
    // 听
    Listen(string) string
    // 打断、插嘴
    Interrupt(string)
}

// 王兰讲师
type WangLan struct {
    msg string
}

func (this *WangLan) Say(msg string) {
    fmt.Printf("王兰说：%s\n", msg)
}

func (this *WangLan) Listen(msg string) string {
    this.msg = msg
    return msg
}

func (this *WangLan) Interrupt(msg string) {
    this.Say(msg)
}

// 江娄讲师
type JiangLou struct {
    msg string
}

func (this *JiangLou) Say(msg string) {
    fmt.Printf("江娄说：%s\n", msg)
}

func (this *JiangLou) Listen(msg string) string {
    this.msg = msg
    return msg
}

func (this *JiangLou) Interrupt(msg string) {
    this.Say(msg)
}

func main() {
    wl := &WangLan{}
    jl := &JiangLou{}

    var person Speaker
    person = wl
    person.Say("Hello World!")
    person = jl
    person.Say("Good Luck!")
}
```

Speaker 接口有两个实现 WangLan 类型和 JiangLou 类型。但是具体到实例来说，变量 wl 和变量 jl 只有是对应实例的指针类型才真正 能被 Speaker 接口变量所持有。这是因为 WangLan 类型和 JiangLou 类型所有对 Speaker 接口的实现都是在 * T 上。这就是上例中 person 能够持有 wl 和 jl 的原因。

想象一下 java 的泛型 (很可惜 golang 不支持泛型)，java 在支持泛型之前需要手动装箱和拆箱。由于 golang 能将不同的类型存入到接口 类型的变量中，使得问题变得更加复杂。所以有时候我们不得不面临这样一个问题：我们究竟往接口存入的是什么样的类型？有没有办法反向查询？答案是肯定的。

Comma-ok 断言的语法是：value, ok := element.(T)。element 必须是接口类型的变量，T 是普通类型。如果断言失败，ok 为 false，否则 ok 为 true 并且 value 为变量的值。来看个例子：

```golang
package main

import (
    "fmt"
)

type Html []interface{}

func main() {
    html := make(Html, 5)
    html[0] = "div"
    html[1] = "span"
    html[2] = []byte("script")
    html[3] = "style"
    html[4] = "head"
    for index, element := range html {
        if value, ok := element.(string); ok {
            fmt.Printf("html[%d] is a string and its value is %s\n", index, value)
        } else if value, ok := element.([]byte); ok {
            fmt.Printf("html[%d] is a []byte and its value is %s\n", index, string(value))
        }
    }
}
```
其实 Comma-ok 断言还支持另一种简化使用的方式：value := element.(T)。但这种方式不建议使用，因为一旦 element.(T) 断言失败，则会产生运行时错误。如：

```golang
package main

import (
    "fmt"
)

func main() {
    var val interface{} = "good"
    fmt.Println(val.(string))
    // fmt.Println(val.(int))
}
```

以上的代码中被注释的那一行会运行时错误。这是因为 val 实际存储的是 string 类型，因此断言失败。

还有一种转换方式是 switch 测试。既然称之为 switch 测试，也就是说这种转换方式只能出现在 switch 语句中。可以很轻松的将刚才用 Comma-ok 断言的例子换成由 switch 测试来实现：

```golang
package main

import (
    "fmt"
)

type Html []interface{}

func main() {
    html := make(Html, 5)
    html[0] = "div"
    html[1] = "span"
    html[2] = []byte("script")
    html[3] = "style"
    html[4] = "head"
    for index, element := range html {
        switch value := element.(type) {
        case string:
            fmt.Printf("html[%d] is a string and its value is %s\n", index, value)
        case []byte:
            fmt.Printf("html[%d] is a []byte and its value is %s\n", index, string(value))
        case int:
            fmt.Printf("invalid type\n")
        default:
            fmt.Printf("unknown type\n")
        }
    }
}
```
