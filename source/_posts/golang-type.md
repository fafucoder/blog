---
title: golang类型别名和类型定义
date: 2020-08-06 14:50:53
tags:
- golang
categories:
- golang
---

### 类型定义
根据基本类型声明一个新的数据类型。

类型定义格式类似如下：
```
type hello int
type world func(name string) string
```

### 类型别名
类型别名 是 Go 1.9 版本添加的新功能。主要应用于代码升级、工程重构、迁移中类型的兼容性问题。C/C++ 语言中，代码的重构升级可以使用宏快速定义新的代码。

类型别名格式类似如下:
```
type hello = int
type world = int32
```

### 类型定义与类型别名
类型别名规定：Type Alias只是Type 的别名，本质上Type Alias 与Type是同一个类型，即基本数据类型是一致的。这意味着这两种类型的数据可以互相赋值，而类型定义要和原始类型赋值的时候需要类型转换(Conversion T(x))。

```
package main

import "fmt"

type world func(name string) string

type hello = int

func main() {
	var h hello
	var w world

	fmt.Printf("hello Type: %T, value： %d\n", h, h)
	fmt.Printf("world Type: %T, value： %d\n", w, w)
}

result:

hello Type: int, value： 0
world Type: main.world, value： 0
```

### 参考文档
- https://blog.csdn.net/AMimiDou_212/article/details/94873163
- https://sanyuesha.com/2017/07/27/go-type/
- https://colobu.com/2017/06/26/learn-go-type-aliases/