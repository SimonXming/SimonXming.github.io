---
layout: post
title: golang deepcopy -- 复杂结构体的指针变量的深拷贝
category: 语言
tags: golang pointer
keywords: golang
description:
---

```go
package main

import (
    "fmt"
    "bytes"
    "encoding/gob"
    "encoding/json"
    "github.com/mohae/deepcopy"
)

type GDeepCopy struct {
	A map[string]string
	B []string
	D int
}

type DeepCopy struct {
	A map[string]string
	B []string
	C Cc
}

type Cc struct {
	D map[int]*string
}

func main() {
	var cc Cc = Cc{
		D: make(map[int]*string, 0),
	}
	var dc DeepCopy = DeepCopy{
		A: make(map[string]string, 0),
		B: []string{`b`, `c`, `d`},
		C: cc,
	}
	dv := DeepCopy{}
	dv = dc
	fmt.Printf("dc: %p\n", &dc)
	fmt.Printf("dv: %p\n", &dv)
	dv.B = []string{`f`, `g`, `d`}
	fmt.Println(`dc:`, dc)
	fmt.Println(`dv:`, dv)

	fmt.Println(`=============================`)
	var df *DeepCopy = &DeepCopy{
		A: make(map[string]string, 0),
		B: []string{`a`, `b`, `c`},
		C: cc,
	}

	// 1. This just copy pointer
	// dn := &DeepCopy{}
	// dn = df

	// 2. This is shallow copy
	// dn := &DeepCopy{}
	// *dn = *df

	// 3. This is actual deep copy
	dn := deepcopy.Copy(df).(*DeepCopy)

	fmt.Printf("dn: %p\n", dn)
	fmt.Printf("df: %p\n", df)
	dn.B = []string{`f`, `g`, `d`}
	linkStr := "This is string."
	df.C.D[1] = &linkStr
	fmt.Println(`df:`, df)
	fmt.Println(`dn:`, dn)
	fmt.Println(`=============================`)
	// otherMethod(cc, df)
}

func otherMethod(cc Cc, df *DeepCopy) {
    var buf bytes.Buffer
    dg := &DeepCopy{}
    json.NewEncoder(&buf).Encode(*df)
    json.NewDecoder(bytes.NewBuffer(buf.Bytes())).Decode(dg)
    dg.B = []string{`h`, `j`, `k`}
    fmt.Printf("df : %p\n", df)
    fmt.Printf("dg: %p\n", dg)
    fmt.Println(`df:`, df)
    fmt.Println(`dg:`, dg)
    fmt.Println(`=============================`)
    dj := &GDeepCopy{}
    gob.NewEncoder(&buf).Encode(*df)
    gob.NewDecoder(bytes.NewBuffer(buf.Bytes())).Decode(dj)
    dj.B = []string{`q`, `w`, `e`}
    fmt.Printf("df : %p\n", df)
    fmt.Printf("dj: %p\n", dj)
    fmt.Println(`df:`, df)
    fmt.Println(`dj:`, dj)
    fmt.Println(`=============================`)
}

/*
=============================
dc: 0xc4200149f0
dv: 0xc420014a80
dc: {map[] [b c d] {map[]}}
dv: {map[] [f g d] {map[]}}
=============================
1. This just copy pointer
dn: 0xc420014b70
df: 0xc420014b70
df: &{map[] [f g d] {map[1:0xc42000e720]}}
dn: &{map[] [f g d] {map[1:0xc42000e720]}}
=============================
2. This is shallow copy
dn: 0xc420014bd0
df: 0xc420014b70
df: &{map[] [a b c] {map[1:0xc42000e730]}}
dn: &{map[] [f g d] {map[1:0xc42000e730]}}
=============================
3. This is actual deep copy
dn: 0xc420014bd0
df: 0xc420014b70
df: &{map[] [a b c] {map[1:0xc42000e740]}}
dn: &{map[] [f g d] {map[]}}
=============================
*/

/*
Other method:
=============================
df : 0xc42007cab0
dg: 0xc42007cbd0
df: &{map[] [f g d] {map[1:0xc420076620]}}
dg: &{map[] [h j k] {map[1:0xc420076890]}}
=============================
df : 0xc42007cab0
dj: 0xc42007cff0
df: &{map[] [f g d] {map[1:0xc420076620]}}
dj: &{map[] [q w e] 0}
=============================
*/
```