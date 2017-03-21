---
layout: post
title: Golang Reflect 的使用介绍
category: 语言
tags: golang fundamental
keywords: reflect
description:
---

## reflect

reflect 库的 godoc 在 http://golang.org/pkg/reflect/

reflect 包有两个数据类型我们必须知道，一个是 Type，一个是 Value。

Type 就是定义的类型的一个数据类型，Value 是值的类型

```golang
package main
import (
    "fmt"
    "reflect"
)
type MyStruct struct {
    name string
}
func (this *MyStruct) GetName(str string) string {
    this.name = str
    return this.name
}
func main() {
    // 备注: reflect.Indirect -> 如果是指针则返回 Elem()
    // 首先，reflect包有两个数据类型我们必须知道，一个是Type，一个是Value。
    // Type就是定义的类型的一个数据类型，Value是值的类型
    // 对象
    s := "this is string"
    // 获取对象类型 (string)
    fmt.Println(reflect.TypeOf(s))
    // 获取对象值 (this is string)
    fmt.Println(reflect.ValueOf(s))
    // 对象
    var x float64 = 3.4
    // 获取对象值 (<float64 Value>)
    fmt.Println(reflect.ValueOf(x))
    // 对象
    a := &MyStruct{name: "nljb"}
    // 返回对象的方法的数量 (1)
    fmt.Println(reflect.TypeOf(a).NumMethod())
    // 遍历对象中的方法
    for m := 0; m < reflect.TypeOf(a).NumMethod(); m++ {
        method := reflect.TypeOf(a).Method(m)
        fmt.Println(method.Type)         // func(*main.MyStruct) string
        fmt.Println(method.Name)         // GetName
        fmt.Println(method.Type.NumIn()) // 参数个数
        fmt.Println(method.Type.In(1))   // 参数类型
    }
    // 获取对象值 (<*main.MyStruct Value>)
    fmt.Println(reflect.ValueOf(a))
    // 获取对象名称
    fmt.Println(reflect.Indirect(reflect.ValueOf(a)).Type().Name())
    // 参数
    i := "Hello"
    v := make([]reflect.Value, 0)
    v = append(v, reflect.ValueOf(i))
    // 通过对象值中的方法名称调用方法 ([nljb]) (返回数组因为Go支持多值返回)
    fmt.Println(reflect.ValueOf(a).MethodByName("GetName").Call(v))
    // 通过对值中的子对象名称获取值 (nljb)
    fmt.Println(reflect.Indirect(reflect.ValueOf(a)).FieldByName("name"))
    // 是否可以改变这个值 (false)
    fmt.Println(reflect.Indirect(reflect.ValueOf(a)).FieldByName("name").CanSet())
    // 是否可以改变这个值 (true)
    fmt.Println(reflect.Indirect(reflect.ValueOf(&(a.name))).CanSet())
    // 不可改变 (false)
    fmt.Println(reflect.Indirect(reflect.ValueOf(s)).CanSet())
    // 可以改变
    // reflect.Indirect(reflect.ValueOf(&s)).SetString("jbnl")
    fmt.Println(reflect.Indirect(reflect.ValueOf(&s)).CanSet())
}
获取 Struct 对象的 Tag
 type Home struct {
    i int `nljb:"100"`
}
func main() {
    home := new(Home)
    home.i = 5
    rcvr := reflect.ValueOf(home)
    typ := reflect.Indirect(rcvr).Type()
    fmt.Println(typ.Kind().String())
    x := typ.NumField()
    for i := 0; i < x; i++ {
        nljb := typ.Field(0).Tag.Get("nljb")
        fmt.Println(nljb)
    }
}
```

### 反射使用案例

```golang
package server
import (
    "fmt"
    "net/http"
    "reflect"
    "strings"
)
type Server struct {
    name    string
    rcvr    reflect.Value
    typ     reflect.Type
    methods map[string]*Method
}
type Method struct {
    method reflect.Method
    json   bool
}
func NewServer() *Server {
    server := new(Server)
    server.methods = make(map[string]*Method)
    return server
}
func (this *Server) Start(port string) error {
    return http.ListenAndServe(port, this)
}
func (this *Server) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    for mname, mmethod := range this.methods {
        if strings.ToLower("/"+this.name+"."+mname) == r.URL.Path {
            if mmethod.json {
                returnValues := mmethod.method.Func.Call(
                    []reflect.Value{this.rcvr, reflect.ValueOf(w), reflect.ValueOf(r)})
                content := returnValues[0].Interface()
                if content != nil {
                    w.WriteHeader(500)
                    ...
                }
            } else {
                mmethod.method.Func.Call(
                    []reflect.Value{this.rcvr, reflect.ValueOf(w), reflect.ValueOf(r)})
            }
        }
    }
}
/*
    func (this *Hello) JsonHello(r *http.Request) {}
    func (this *Hello) Hello(w http.ResponseWriter, r *http.Request) {}
*/
func (this *Server) Register(rcvr interface{}) error {
    this.typ = reflect.TypeOf(rcvr)
    this.rcvr = reflect.ValueOf(rcvr)
    this.name = reflect.Indirect(this.rcvr).Type().Name()
    if this.name == "" {
        return fmt.Errorf("no service name for type ", this.typ.String())
    }
    for m := 0; m < this.typ.NumMethod(); m++ {
        method := this.typ.Method(m)
        mtype := method.Type
        mname := method.Name
        if strings.HasPrefix(mname, "Json") {
            if mtype.NumIn() != 2 {
                return fmt.Errorf("method %s has wrong number of ins: %d", mname, mtype.NumIn())
            }
            arg := mtype.In(1)
            if arg.String() != "*http.Request" {
                return fmt.Errorf("%s argument type not exported: %s", mname, arg)
            }
            this.methods[mname] = &Method{method, true}
        } else {
            if mtype.NumIn() != 3 {
                return fmt.Errorf("method %s has wrong number of ins: %d", mname, mtype.NumIn())
            }
            reply := mtype.In(1)
            if reply.String() != "http.ResponseWriter" {
                return fmt.Errorf("%s argument type not exported: %s", mname, reply)
            }
            arg := mtype.In(2)
            if arg.String() != "*http.Request" {
                return fmt.Errorf("%s argument type not exported: %s", mname, arg)
            }
            this.methods[mname] = &Method{method, false}
        }
    }
    return nil
}
// ... //
type Hello struct {
}
func (this *Hello) Print(w http.ResponseWriter, r *http.Request) map[string]interface{} {
    w.Write([]byte("print"))
    return nil
}
func (this *Hello) Hello(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("hello"))
}
func (this *Hello) JsonHello(r *http.Request) {
}
func main() {
    server := NewServer()
    fmt.Println(server.Register(new(Hello)))
    server.Start(":8080")
}
```