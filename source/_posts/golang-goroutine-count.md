---
title: goroutine并发数量控制
date: 2021-11-09 10:02:02
tags:
- golang
categories:
- golang
---

### 概述

在golang语言中创建协程（Goroutine）的成本非常低，因此稍不注意就可能创建出大量的协程，一方面会造成资源的浪费，例如有一万个任务需要处理，如果启用一万个goroutine同时处理，意味了CPU内存资源大量的飙升，所以一般会控制goroutine的数量，例如最多只有一百个goroutine在运行，本章将看下如何控制goroutine的并发数量

### goroutine并发控制

##### 并发未控制情形

在说明goroutine并发控制前，先看下并发不控制的代码逻辑

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	wg := sync.WaitGroup{}
	workerCount := 100
	for i := 0; i < workerCount; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			fmt.Printf("i am goroutine %d \n", i)
		}(i)
	}

	wg.Wait()
	fmt.Println("worker have done")

}
```

上面的输出如下, 在多核的场景下，goroutine不一定是顺序输出的:

```
......
i am goroutine 88 
i am goroutine 63 
i am goroutine 89 
i am goroutine 98 
i am goroutine 82 
i am goroutine 93 
i am goroutine 87 
i am goroutine 95 
i am goroutine 97 
worker have done
```

下图展示了为每个 job 创建一个 goroutine 的情况（换句话说，goroutine 的数量是不受控制的）。此种情况虽然生成了很多的 goroutine，但是每个 CPU 核上同一时间只能执行一个 goroutine；当 job 很多且生成了相应数目的 goroutine 后，会出现很多等待执行的 goroutine，从而造成资源上的浪费。

![goroutine并发不控制](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gwa9ewaz1uj31360hijso.jpg)

##### 并发控制概述

给每个 job 生成一个 goroutine 的方式显得粗暴了很多，那么可以通过什么样的方式控制 goroutine 的数目呢？其实上面的代码通过一个 for-range 循环完成了两件事情：①为每个 job 创建 goroutine；②把任务相关的标识传给相应的 goroutine 执行。为了控制 goroutine 的数目，完全可以把上面的两个过程拆分开：a）先通过一个 for-range 循环创建指定数目的 goroutine，b）然后通过 channel/buffered channel 给每个 goroutine 传递任务相关的信息（这里的channel是否缓冲无所谓，主要用到的是 channel 的线程安全特性）。如下图所示。

![goroutine并发控制](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008i3skNly1gwaa7xeiyrj313q0i2wgn.jpg)

##### goroutine并发控制方案一

针对上面的代码，如果想达到goroutine并发执行的控制，我们可以加个buffer channel来限制最多只有多少个goroutine在执行，代码如下：

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	taskCount := 10
	worker := make(chan int, 3)
	wg := sync.WaitGroup{}

	for i := 0; i < taskCount; i++ {
		wg.Add(1)

		go func(i int) {
			defer wg.Done()

			worker <- i
			fmt.Println("i am worker ", i)
			<-worker
		}(i)
	}

	wg.Wait()
}
```

通过channel限制最多有三个goroutine在执行，其余的被挂起等待中，但是此方案也有个缺陷，那就是goroutine还是都被创建了，只不过这些goroutine被挂起了而已。

##### goroutine并发控制方案二

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	taskCount, workerCount := 10, 3
	worker := make(chan int, workerCount)
	wg := sync.WaitGroup{}

	for i := 0; i < workerCount; i++ {
		go func() {
			for w := range worker {
				fmt.Println("i am worker ", w)
				wg.Done()
			}
		}()
	}

	for i := 0; i < taskCount; i++ {
		wg.Add(1)
		worker <- i
	}

	wg.Wait()
}
```

在方案二中，我们起了三个goroutine一直去消费woker以达到限制goroutine最大并发数的目的。

### 参考文档

- [【图示】控制 Goroutine 的并发数量的方式](https://jingwei.link/2019/09/13/conotrol-goroutines-count.html    )   // jing 维大佬
- [来，控制一下 goroutine 的并发数量](https://eddycjy.com/posts/go/talk/2019-01-20-control-goroutine/)        // 煎鱼老师