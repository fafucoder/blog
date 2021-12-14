---
title: golang-context
date: 2021-02-08 10:18:50
tags:
- golang
categories:
- golang
---

### context 作用

context用来解决goroutine 之间传递上下文信息，包括：取消信号、超时时间、截止时间、k-v 等。

```go


package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	ctx, cel := context.WithCancel(context.Background())

	go pushDockerToHarbor(ctx)
	go pushImageToMinio(ctx)

	time.Sleep(6 * time.Second)
	cel()

	time.Sleep(5 * time.Second)
}

func pushImageToMinio(ctx context.Context) {
	go func() {
		fmt.Println("i doing push image to minio worker")
		time.Sleep(5 * time.Second)
	}()

	for {
		select {
		case <-ctx.Done():
			fmt.Println("canceled")
			return
		case <-time.After(30 * time.Minute):
			fmt.Println("timeout")
			return
		}
	}
}

func pushDockerToHarbor(ctx context.Context) {
	go func() {
		fmt.Println("i doing push docker to harbor worker")
		time.Sleep(5 * time.Second)
	}()

	for {
		select {
		case <-ctx.Done():
			fmt.Println("canceled")
			return
		case <-time.After(30 * time.Minute):
			fmt.Println("timeout")
			return
		}
	}
}

```

### 参考文档

- [深度解密Go语言之context](https://www.qcrao.com/2019/06/12/dive-into-go-context/)