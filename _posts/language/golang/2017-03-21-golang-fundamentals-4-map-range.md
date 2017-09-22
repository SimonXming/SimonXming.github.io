---
layout: post
title: Golang fundamental 4 map range
category: 语言
tags: golang fundamental
keywords: map range
description:
---

## Map

map 是 Golang 中的一种 Associative data type。提供类似于其他语言中 hash 或者 dictionary 的功能。map 类型是 Golang 语言中 3 个引用类型（slice， map， channel）之一。map 对象中的 Keys 元素必须不重复而且属于同一类型，该类型必须支持 == 操作符，可以用于比较；Values 元素也必须属于同一类型，该类型可以是任意类型。

`map` 对象的声明形式为 `map[TypeOfKey]TypeOfValue`。

还是从例子入手：

```golang
package main

import "fmt"

func main() {
	// 如果已经知道 map 中的数据，可以直接以下面的方式声明 map 对象
	v_map := map[int]string{
		1: "One",
		2: "Two",
		3: "Three",
	}
	fmt.Println("v_map: ", v_map)
	fmt.Println("len(v_map): ", len(v_map))

	// 从下面的用法可以看到 “修改已有值” 和 “添加新 pair” 形式上是一样的。
	// 如果 key 值已存在，就是修改操作，否则就是添加新 pair
	// 修改已有值
	v_map[1] = "One hundred"

	// 添加新 pair
	v_map[4] = "Two hundred"
	fmt.Println("After changes, v_map: ", v_map)
	fmt.Println("After changes, len(v_map): ", len(v_map))

	// 如果只是声明一个 map 对象，可以使用内置函数 make
	v_mapByMake := make(map[int]string)
	v_mapByMake[0] = "One hundred"
	fmt.Println("make(map[int]string, 10)，v_mapByMake: ", v_mapByMake)

	// 使用 make 时也可以设定预期的键值对数量，在初始化时一次性分配大量内存，从而避免使用过程中频繁动态分配
	// 这里给定的数量值不会影响初始化后 len(mapObject)
	v_mapByMake = make(map[int]string, 10)
	fmt.Println("make(map[int]string, 10)，Before chagnes，v_mapByMake: ", v_mapByMake)
	fmt.Println("make(map[int]string, 10)，Before chagnes，len(v_mapByMake): ", len(v_mapByMake))
	v_mapByMake[0] = "One hundred"
	v_mapByMake[1] = "Two hundred"
	v_mapByMake[2] = "Three hundred"
	fmt.Println("make(map[int]string, 10)，After chagnes，v_mapByMake: ", v_mapByMake)

	// 遍历 map 对象中的键值对
	// 注意，这里迭代的顺序是不确定的
	for key, value := range v_mapByMake {
		fmt.Printf("%d : %s\n", key, value)
	}

	// 检查 map 对象中是否存在某个 key 索引的元素，如果存在获取该 key 索引的 value
	if value, ok := v_mapByMake[1]; ok {
		fmt.Println("value, ok := v_mapByMake[1]，value: ", value)
	}

	// 这里如果只是判断是否存在，可以使用占位符
	_, ok := v_mapByMake[1]
	fmt.Println("_, ok := v_mapByMake[1]，ok: ", ok)

	// 如果尝试获取不存在的元素，会返回空，不会抛出异常
	fmt.Println("v_mapByMake[10]: ", v_mapByMake[10])

	// 如果尝试删除不存在的元素，对已有数据不会有影响，不会抛出异常
	fmt.Println("Before deletion non-existed elem, v_mapByMake: ", v_mapByMake)
	delete(v_mapByMake, 10)
	fmt.Println("After deletion non-existed elem, v_mapByMake: ", v_mapByMake)

	// 从 map 中获取的 value 是原始数据的拷贝，如果其本身是值类型，对其修改时不允许的。
	// 如果是引用类型，修改就没有问题。
	// 比如下面的例子，如果值是 slice 对象，对其元素的修改是允许的。
	// 如果值是 array 对象，则通过 map 的索引获取到的 value 是不允许修改的。
	// 将下面 map 的元素类型修改为 array 类型 map[int][3]int, 尝试修改会导致编译错误
	v_mapOfArray := map[int][]int{
		1: {0, 1, 2},
		2: {3, 4, 5},
	}
	fmt.Println("Before change, v_mapOfArray is: ", v_mapOfArray)
	fmt.Println("Before change, v_mapOfArray[1][0] is: ", v_mapOfArray[1][0])
	v_mapOfArray[1][0] = 100
	fmt.Println("After change, v_mapOfArray[1][0] is: ", v_mapOfArray[1][0])
	fmt.Println("After change, v_mapOfArray is: ", v_mapOfArray)

	v_map = map[int]string{
		1: "One",
		2: "Two",
		3: "Three",
	}

	fmt.Println("Before iterating, v_map: ", v_map)
	// 迭代过程中可以安全地删除键值对，在迭代过程中也支持添加新的键值对
	for key, _ := range v_map {
		delete(v_map, key)
		if key == 1 {
			v_map[key*5] = "New"
		}
	}
	fmt.Println("After iterating, v_map: ", v_map)

}
```

将上面的代码存入源文件 map.go 并使用 go run map.go 可以看到下面的输入：

```shell
v_map:  map[1:One 2:Two 3:Three]
len(v_map):  3
After changes, v_map:  map[1:One hundred 2:Two 3:Three 4:Two hundred]
After changes, len(v_map):  4
make(map[int]string, 10)，v_mapByMake:  map[0:One hundred]
make(map[int]string, 10)，Before chagnes，v_mapByMake:  map[]
make(map[int]string, 10)，Before chagnes，len(v_mapByMake):  0
make(map[int]string, 10)，After chagnes，v_mapByMake:  map[0:One hundred 1:Two hundred 2:Three hundred]
2 : Three hundred
0 : One hundred
1 : Two hundred
value, ok := v_mapByMake[1]，value:  Two hundred
_, ok := v_mapByMake[1]，ok:  true
v_mapByMake[10]:  
Before deletion non-existed elem, v_mapByMake:  map[2:Three hundred 0:One hundred 1:Two hundred]
After deletion non-existed elem, v_mapByMake:  map[2:Three hundred 0:One hundred 1:Two hundred]
Before change, v_mapOfArray is:  map[1:[0 1 2] 2:[3 4 5]]
Before change, v_mapOfArray[1][0] is:  0
After change, v_mapOfArray[1][0] is:  100
After change, v_mapOfArray is:  map[1:[100 1 2] 2:[3 4 5]]
Before iterating, v_map:  map[1:One 2:Two 3:Three]
After iterating, v_map:  map[5:New]
```

注意，官方文档 中对使用内置函数 make 创建 map 对象时这样说：

```shell
make(T)          map        map of type T
make(T, n)       map        map of type T with initial space for n elements
```

从中可以看到，我们可以在创建 map 对象时设定预期的元素个数，这样在 map 对象中的键值对增加时避免频繁动态分配内存，提高效率。

## Range

官网 For statement 对 range 表达式的返回值做了如下说明：

```shell
Range expression                          1st value          2nd value

array or slice  a  [n]E, *[n]E, or []E    index    i  int    a[i]       E
string          s  string type            index    i  int    see below  rune
map             m  map[K]V                key      k  K      m[k]       V
channel         c  chan E, <-chan E       element  e  E
```

如果 a 是一个数组、数组的指针或者切片（slice），产生的索引以递增的顺序创建，如果索引是 for 语句的参数，则其从 0 开始到 len(a) - 1。对于空 slice 来说，索引总数为零。

对于 string 类型的 a，range 从该字符串中首个字符开始，每次增加一个 Unicode code points。在连续的迭代过程中，index 的索引为字符串经过 UTF-8 编码后连续的 code points 的首个字节索引，rune 类型的 range 返回的第二个值就是当前 UTF-8 code point 的值。如果迭代过程遇到非法的 UTF-8 编码，则 range 返回的第二个值规定为 0xFFFD，一个 Unicode 替代字符，然后迭代器向前前进一个字节。

map 中的索引顺序没有定义（not specified），Golang 不保证使用不同的索引后结果的顺序相同。如果 map 中的 pair 还未被访问时被删除了，该 pair 对应的 iteration 就不会生成。如果在迭代过程中创建新的键值对，可以选择是否在哪个具体的迭代过程中来创建。如果 map 对象是空的，则索引总数为零。

对于 channel 对象来说，range 产生的索引的值来自这个发送到该 channel 对象的连续的值，直到该对象被关闭（closed）。如果对象为 nil，range 表达式会被一直 block。