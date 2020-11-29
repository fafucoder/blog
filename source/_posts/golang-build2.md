---
title: golang build 调试技巧
date: 2020-11-29 17:35:17
tags:
- golang
categories:
- golang
---

### 编译技巧

#### go build mod

使用 `go build -mod=vendor` 来构建项目，因为在 `go modules` 模式下 `go build` 是屏蔽 vendor 机制的，所以需要特定参数重新开启 vendor 机制, 这样子断点调试的时候，就能够在vendor文件夹下断点了。如果golang的版本大于等于1.14, 则默认使用-mod=vendor

![go mod](https://tva1.sinaimg.cn/large/0081Kckwly1gl66qxf5wzj31fa0dwq7d.jpg)

#### go build tag

-ldflags 可以传递指令,也可以通过`// +build` 标记去做，如下所示：

```go
// +build release

package main

const version = "RELEASE"
```

上面代码的关键是 `// +build release`这行代码，注意这行代码前后须有一个空行隔开，例如在第一行时，接下来要空出一行。这个文件只会被go bulid识别到，而go run等命令不会去识别这个文件，而且vscode, golang等编辑器也会略过这个文件。

通过引入 //  +build的好处就是可以作为文件因为包， 比如main包要引入其他main 包的方案中，可以创建一个空的文件,然后引入文件。

```go
// +build tools

package lib

import (
	_ "k8s.io/kube-openapi/cmd/openapi-gen"
)

```

### 参考文档

- [go mod refer](https://golang.org/ref/mod)
- [go build -tags 使用](https://www.cnblogs.com/linyihai/p/10859945.html)