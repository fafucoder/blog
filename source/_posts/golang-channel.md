---
title: golang channel解析
date: 2021-01-16 22:55:20
tags:
- golang
categories:
- golang
---

### channel的基本概念

#### 1. channel的基本概念

channel，通道。golang中用于数据传递的一种数据结构， 是golang中一种传递数据的方式，常用于goroutine之间的通信。 像管道一样，一个goroutineA向channelA中放数据，另外一个goroutineB从channelA中取数据。

#### 2. channel的申明、传值、关闭

channel是指针类型的数据类型，可通过make来分配内存。例如：`ch := make(chan int)`，这表示创建一个channel，这个channel中只能保存int类型的数据。也就是说一端只能向此channel中放进int类型的值，另一端只能从此channel中读出int类型的值。

使用chan关键字声明一个通道，在使用前必须先创建，操作符 `<-` 用于指定通道的方向，发送或接收。如果未指定方向，则为双向通道。

```go
//声明和创建
var ch chan int      // 声明一个传递int类型的channel
ch := make(chan int) // 使用内置函数make()定义一个channel
ch2 := make(chan interface{})         // 创建一个空接口类型的通道, 可以存放任意格式

type Equip struct{ /* 一些字段 */ }
ch2 := make(chan *Equip)             // 创建Equip指针类型的通道, 可以存放*Equip

//传值
ch <- value          // 将一个数据value写入至channel，这会导致阻塞，直到有其他goroutine从这个channel中读取数据
value := <-ch        // 从channel中读取数据，如果channel之前没有写入数据，也会导致阻塞，直到channel中被写入数据为止

ch := make(chan interface{})  // 创建一个空接口通道
ch <- 0 // 将0放入通道中
ch <- "hello"  // 将hello字符串放入通道中

//关闭
close(ch)            // 关闭channel
```

#### 3. buffer channel和unbuffer channel

channel分为两种：unbuffered channel和buffered channel

- unbuffered channel：阻塞、同步模式

  - sender端向channel中send一个数据，然后阻塞，直到receiver端将此数据receive
  - receiver端一直阻塞，直到sender端向channel发送了一个数据

  ```go
  package main
  
  import (
  	"fmt"
  )
  
  var done chan bool
  
  func HelloWorld() {
  	fmt.Println("Hello world goroutine")
  	done <- true
  }
  func main() {
  	done = make(chan bool) // 创建一个channel
  	go HelloWorld()
  	<-done
  }
  
  // output "hello world goroutine"
  ```

- buffered channel：非阻塞、异步模式

  - sender端可以向channel中send多个数据(只要channel容量未满)，容量满之前不会阻塞
  - receiver端按照队列的方式(FIFO,先进先出)从buffered channel中按序receive其中数据

  ```go
  	ch := make(chan string, 3) // 创建了缓冲区为3的通道
  	fmt.Println(len(ch))
  	fmt.Println(cap(ch))
  ```

#### 4. channel foreach(遍历)

可以通过for无限循环来读取channel中的数据，但是可以使用range来迭代channel， 它会返回每次迭代过程中所读取的数据，直到channel被关闭。

> **注意： 只要channel未关闭，range迭代channel就会一直被阻塞。**

```go
channel := make(chan int, 10)
func printChannel1() {
	for {
		c := <-channel
		fmt.Println(result)
	}
}

func printChannel2(count chan int) {
	for c := range count {
		fmt.Println(c)
	}
}
```

#### 5. channel死锁

把数据往通道中发送时，如果接收方一直都没有接收，那么发送操作将持续阻塞。Go 程序运行时能智能地发现一些永远无法发送成功的语句并报错, 例如以下情形将将出现deadlock:

> **只要所有goroutine都被阻塞(包括main函数)，就会出现死锁**

```go
fatal error: all goroutines are asleep - deadlock! 
//运行时发现所有的 goroutine（包括main）都处于等待 goroutine。
```

实例一：

```go
package main

import "fmt"

func main() {
	ch := make(chan string)
	ch <- "hello world"
	fmt.Println(<-ch)
}
```

执行这段代码后将会出现如下错误：

```go
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
        /Users/xxx/go/src/examples/worker_pool/main.go:7 +0x59

Process finished with exit code 2
```

出现deadlock的原因是因为`ch<-"hello world"`这行导致了main goroutine被挂起，导致`fmt.Println(<ch)`这一行得不到执行，且没有任何其他goroutine去消费这个channel, 因此造成了deadlock。

示例二：

```go
package main

import "fmt"

func main() {
	ch := make(chan string)
	go func() {
		fmt.Println(<-ch)
	}()
	ch <- "hello"
	ch <- "world"
}
```

执行完这段代码后，也会出现死锁的情况如下：

```go
hello
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
        /Users/dawn/go/src/examples/worker_pool/main.go:11 +0x90

Process finished with exit code 2
```

出现这样的结果是因为通道实际上是类型化消息的队列，它是先进先出(`FIFO`)的结构，可以保证发送给它们的元素的顺序。所以上面代码只取出了第一次传的值，即"hello"，而第二次传入的值没有一个配对的接收者来接收，因此就出现了`deadlock`。

示例三：

```go
package main

import "fmt"

func main() {
	ch := make(chan string)
	go func() {
		ch <- "hello"
		ch <- "world"
	}()

	for {
		fmt.Printf("%s \n", <-ch)
	}
}
```

执行完这段代码也会出现deadlock的情况如下：

```go
hello 
world 
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive]:
main.main()
        /Users/dawn/go/src/examples/worker_pool/main.go:13 +0x7c

Process finished with exit code 2
```

出现上面的结果是因为`for`循环一直在获取通道中的值，但是在读取完 `hello和`world后，通道中没有新的值传入，这样接收者就阻塞了。

### channel原理解析

channel的结构如下所示(channel本质是消息传递的数据结构):

```go
type hchan struct {
	// chan 里元素数量
	qcount   uint
	// chan 底层循环数组的长度
	dataqsiz uint
	// 指向底层循环数组的指针
	// 只针对有缓冲的 channel
	buf      unsafe.Pointer
	// chan 中元素大小
	elemsize uint16
	// chan 是否被关闭的标志
	closed   uint32
	// chan 中元素类型
	elemtype *_type // element type
	// 已发送元素在循环数组中的索引
	sendx    uint   // send index
	// 已接收元素在循环数组中的索引
	recvx    uint   // receive index
	// 等待接收的 goroutine 队列
	recvq    waitq  // list of recv waiters
	// 等待发送的 goroutine 队列
	sendq    waitq  // list of send waiters

	// 保护 hchan 中所有字段
	lock mutex
}
```

- `buf` :指向底层循环数组，只有缓冲型的 channel 才有。
- `sendx`，`recvx` :指向底层循环数组，表示当前可以发送和接收的元素位置索引值（相对于底层数组）
- `sendq`，`recvq` :表示被阻塞的 goroutine，这些 goroutine 由于尝试读取 channel 或向 channel 发送数据而被阻塞。
- `lock` :用来保证每个读 channel 或写 channel 的操作都是原子的。

waitq` 是 `sudog` 的一个双向链表，而 `sudog` 实际上是对 goroutine 的一个封装：

```go
type waitq struct {
	first *sudog
	last  *sudog
}
```

例如，创建一个容量为 6 的，元素为 int 型的 channel 数据结构如下 ：

![channel](https://tva1.sinaimg.cn/large/008i3skNly1gwaf4ufem0j312k0p6abi.jpg)

#### 创建channel

创建channel 在底层使用的是函数`makechan`:

```go
func makechan(t *chantype, size int64) *hchan
```

从函数原型来看，创建的 chan 是一个指针。所以我们能在函数间直接传递 channel，而不用传递 channel 的指针。

具体代码如下：

```go
const hchanSize = unsafe.Sizeof(hchan{}) + uintptr(-int(unsafe.Sizeof(hchan{}))&(maxAlign-1))

func makechan(t *chantype, size int64) *hchan {
	elem := t.elem

	// 省略了检查 channel size，align 的代码
	// ……

	var c *hchan
	// 如果元素类型不含指针 或者 size 大小为 0（无缓冲类型）
	// 只进行一次内存分配
	if elem.kind&kindNoPointers != 0 || size == 0 {
		// 如果 hchan 结构体中不含指针，GC 就不会扫描 chan 中的元素
		// 只分配 "hchan 结构体大小 + 元素大小*个数" 的内存
		c = (*hchan)(mallocgc(hchanSize+uintptr(size)*elem.size, nil, true))
		// 如果是缓冲型 channel 且元素大小不等于 0（大小等于 0的元素类型：struct{}）
		if size > 0 && elem.size != 0 {
			c.buf = add(unsafe.Pointer(c), hchanSize)
		} else {
			// race detector uses this location for synchronization
			// Also prevents us from pointing beyond the allocation (see issue 9401).
			// 1. 非缓冲型的，buf 没用，直接指向 chan 起始地址处
			// 2. 缓冲型的，能进入到这里，说明元素无指针且元素类型为 struct{}，也无影响
			// 因为只会用到接收和发送游标，不会真正拷贝东西到 c.buf 处（这会覆盖 chan的内容）
			c.buf = unsafe.Pointer(c)
		}
	} else {
		// 进行两次内存分配操作
		c = new(hchan)
		c.buf = newarray(elem, int(size))
	}
	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	// 循环数组长度
	c.dataqsiz = uint(size)

	// 返回 hchan 指针
	return c
}
```

说完channel的结构后，以一个例子说明channel如何接收和发送的

```go
func goroutineA(a <-chan int) {
	val := <- a
	fmt.Println("G1 received data: ", val)
	return
}

func goroutineB(b <-chan int) {
	val := <- b
	fmt.Println("G2 received data: ", val)
	return
}

func main() {
	ch := make(chan int)
	go goroutineA(ch)
	go goroutineB(ch)
	ch <- 3
	time.Sleep(time.Second)
}
```

#### 接收

channel的接收操作有两种写法，一种带 “ok”，反应 channel 是否关闭；一种不带 “ok”，这种写法，当接收到相应类型的零值时无法知道是真实的发送者发送过来的值，还是 channel 被关闭后，返回给接收者的默认类型的零值。两种写法，都有各自的应用场景。

```go
result := <-ch
result, ok := <-ch
```

经过编译器的处理后，这两种写法最后对应源码里的这两个函数：

```go
// entry points for <- c from compiled code
func chanrecv1(c *hchan, elem unsafe.Pointer) {
	chanrecv(c, elem, true)
}

func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
	_, received = chanrecv(c, elem, true)
	return
}
```

`chanrecv1` 函数处理不带 “ok” 的情形，`chanrecv2` 则通过返回 “received” 这个字段来反应 channel 是否被关闭。接收值则比较特殊，会“放到”参数 `elem` 所指向的地址了，这很像 C/C++ 里的写法。如果代码里忽略了接收值，这里的 elem 为 nil。

无论如何，最终转向了 `chanrecv` 函数：

```go
// 位于 src/runtime/chan.go

// chanrecv 函数接收 channel c 的元素并将其写入 ep 所指向的内存地址。
// 如果 ep 是 nil，说明忽略了接收值。
// 如果 block == false，即非阻塞型接收，在没有数据可接收的情况下，返回 (false, false)
// 否则，如果 c 处于关闭状态，将 ep 指向的地址清零，返回 (true, false)
// 否则，用返回值填充 ep 指向的内存地址。返回 (true, true)
// 如果 ep 非空，则应该指向堆或者函数调用者的栈

func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// 省略 debug 内容 …………

	// 如果是一个 nil 的 channel
	if c == nil {
		// 如果不阻塞，直接返回 (false, false)
		if !block {
			return
		}
		// 否则，接收一个 nil 的 channel，goroutine 挂起
		gopark(nil, nil, "chan receive (nil chan)", traceEvGoStop, 2)
		// 不会执行到这里
		throw("unreachable")
	}

	// 在非阻塞模式下，快速检测到失败，不用获取锁，快速返回
	// 当我们观察到 channel 没准备好接收：
	// 1. 非缓冲型，等待发送列队 sendq 里没有 goroutine 在等待
	// 2. 缓冲型，但 buf 里没有元素
	// 之后，又观察到 closed == 0，即 channel 未关闭。
	// 因为 channel 不可能被重复打开，所以前一个观测的时候 channel 也是未关闭的，
	// 因此在这种情况下可以直接宣布接收失败，返回 (false, false)
	if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
		c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
		atomic.Load(&c.closed) == 0 {
		return
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	// 加锁
	lock(&c.lock)

	// channel 已关闭，并且循环数组 buf 里没有元素
	// 这里可以处理非缓冲型关闭 和 缓冲型关闭但 buf 无元素的情况
	// 也就是说即使是关闭状态，但在缓冲型的 channel，
	// buf 里有元素的情况下还能接收到元素
	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(unsafe.Pointer(c))
		}
		// 解锁
		unlock(&c.lock)
		if ep != nil {
			// 从一个已关闭的 channel 执行接收操作，且未忽略返回值
			// 那么接收的值将是一个该类型的零值
			// typedmemclr 根据类型清理相应地址的内存
			typedmemclr(c.elemtype, ep)
		}
		// 从一个已关闭的 channel 接收，selected 会返回true
		return true, false
	}

	// 等待发送队列里有 goroutine 存在，说明 buf 是满的
	// 这有可能是：
	// 1. 非缓冲型的 channel
	// 2. 缓冲型的 channel，但 buf 满了
	// 针对 1，直接进行内存拷贝（从 sender goroutine -> receiver goroutine）
	// 针对 2，接收到循环数组头部的元素，并将发送者的元素放到循环数组尾部
	if sg := c.sendq.dequeue(); sg != nil {
		// Found a waiting sender. If buffer is size 0, receive value
		// directly from sender. Otherwise, receive from head of queue
		// and add sender's value to the tail of the queue (both map to
		// the same buffer slot because the queue is full).
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

	// 缓冲型，buf 里有元素，可以正常接收
	if c.qcount > 0 {
		// 直接从循环数组里找到要接收的元素
		qp := chanbuf(c, c.recvx)

		// …………

		// 代码里，没有忽略要接收的值，不是 "<- ch"，而是 "val <- ch"，ep 指向 val
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		// 清理掉循环数组里相应位置的值
		typedmemclr(c.elemtype, qp)
		// 接收游标向前移动
		c.recvx++
		// 接收游标归零
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		// buf 数组里的元素个数减 1
		c.qcount--
		// 解锁
		unlock(&c.lock)
		return true, true
	}

	if !block {
		// 非阻塞接收，解锁。selected 返回 false，因为没有接收到值
		unlock(&c.lock)
		return false, false
	}

	// 接下来就是要被阻塞的情况了
	// 构造一个 sudog
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}

	// 待接收数据的地址保存下来
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.selectdone = nil
	mysg.c = c
	gp.param = nil
	// 进入channel 的等待接收队列
	c.recvq.enqueue(mysg)
	// 将当前 goroutine 挂起
	goparkunlock(&c.lock, "chan receive", traceEvGoBlockRecv, 3)

	// 被唤醒了，接着从这里继续执行一些扫尾工作
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	closed := gp.param == nil
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, !closed
}
```

- 如果 channel 是一个空值（nil），在非阻塞模式下，会直接返回。在阻塞模式下，会调用 gopark 函数挂起 goroutine，这个会一直阻塞下去。因为在 channel 是 nil 的情况下，要想不阻塞，只有关闭它，但关闭一个 nil 的 channel 又会发生 panic，所以没有机会被唤醒了。
- 和发送函数一样，接下来搞了一个在非阻塞模式下，不用获取锁，快速检测到失败并且返回的操作。顺带插一句，我们平时在写代码的时候，找到一些边界条件，快速返回，能让代码逻辑更清晰，因为接下来的正常情况就比较少，更聚焦了，看代码的人也更能专注地看核心代码逻辑了。

当我们观察到 channel 没准备好接收：

1. 非缓冲型，等待发送列队里没有 goroutine 在等待
2. 缓冲型，但 buf 里没有元素

之后，又观察到 closed == 0，即 channel 未关闭。

因为 channel 不可能被重复打开，所以前一个观测的时候， channel 也是未关闭的，因此在这种情况下可以直接宣布接收失败，快速返回。因为没被选中，也没接收到数据，所以返回值为 (false, false)。

- 接下来的操作，首先会上一把锁，粒度比较大。如果 channel 已关闭，并且循环数组 buf 里没有元素。对应非缓冲型关闭和缓冲型关闭但 buf 无元素的情况，返回对应类型的零值，但 received 标识是 false，告诉调用者此 channel 已关闭，你取出来的值并不是正常由发送者发送过来的数据。但是如果处于 select 语境下，这种情况是被选中了的。很多将 channel 用作通知信号的场景就是命中了这里。
- 接下来，如果有等待发送的队列，说明 channel 已经满了，要么是非缓冲型的 channel，要么是缓冲型的 channel，但 buf 满了。这两种情况下都可以正常接收数据。

于是，调用 recv 函数：

```
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	// 如果是非缓冲型的 channel
	if c.dataqsiz == 0 {
		if raceenabled {
			racesync(c, sg)
		}
		// 未忽略接收的数据
		if ep != nil {
			// 直接拷贝数据，从 sender goroutine -> receiver goroutine
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
		// 缓冲型的 channel，但 buf 已满。
		// 将循环数组 buf 队首的元素拷贝到接收数据的地址
		// 将发送者的数据入队。实际上这时 revx 和 sendx 值相等
		// 找到接收游标
		qp := chanbuf(c, c.recvx)
		// …………
		// 将接收游标处的数据拷贝给接收者
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}

		// 将发送者数据拷贝到 buf
		typedmemmove(c.elemtype, qp, sg.elem)
		// 更新游标值
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.sendx = c.recvx
	}
	sg.elem = nil
	gp := sg.g

	// 解锁
	unlockf()
	gp.param = unsafe.Pointer(sg)
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}

	// 唤醒发送的 goroutine。需要等到调度器的光临
	goready(gp, skip+1)
}
```

如果是非缓冲型的，就直接从发送者的栈拷贝到接收者的栈。否则，就是缓冲型 channel，而 buf 又满了的情形。说明发送游标和接收游标重合了，因此需要先找到接收游标：

```go
func recvDirect(t *_type, sg *sudog, dst unsafe.Pointer) {
	// dst is on our stack or the heap, src is on another stack.
	src := sg.elem
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
	memmove(dst, src, t.size)
}

// chanbuf(c, i) is pointer to the i'th slot in the buffer.
func chanbuf(c *hchan, i uint) unsafe.Pointer {
	return add(c.buf, uintptr(i)*uintptr(c.elemsize))
}
```

将该处的元素拷贝到接收地址。然后将发送者待发送的数据拷贝到接收游标处。这样就完成了接收数据和发送数据的操作。接着，分别将发送游标和接收游标向前进一，如果发生“环绕”，再从 0 开始。

最后，取出 sudog 里的 goroutine，调用 goready 将其状态改成 “runnable”，待发送者被唤醒，等待调度器的调度。

- 然后，如果 channel 的 buf 里还有数据，说明可以比较正常地接收。注意，这里，即使是在 channel 已经关闭的情况下，也是可以走到这里的。这一步比较简单，正常地将 buf 里接收游标处的数据拷贝到接收数据的地址。
- 到了最后一步，走到这里来的情形是要阻塞的。当然，如果 block 传进来的值是 false，那就不阻塞，直接返回就好了。

先构造一个 sudog，接着就是保存各种值了。注意，这里会将接收数据的地址存储到了 `elem` 字段，当被唤醒时，接收到的数据就会保存到这个字段指向的地址。然后将 sudog 添加到 channel 的 recvq 队列里。调用 goparkunlock 函数将 goroutine 挂起。

我们继续之前的例子。前面说到第 14 行，创建了一个非缓冲型的 channel，接着，第 15、16 行分别创建了一个 goroutine，各自执行了一个接收操作。通过前面的源码分析，我们知道，这两个 goroutine （后面称为 G1 和 G2 好了）都会被阻塞在接收操作。G1 和 G2 会挂在 channel 的 recq 队列中，形成一个双向循环链表。因此此时chan的状态如下：

![chan status](https://tva1.sinaimg.cn/large/008i3skNly1gwaflcunbaj31100mw0tn.jpg)

G1 和 G2 被挂起了，状态是 `WAITING`, goroutine 是用户态的协程，由 Go runtime 进行管理，作为对比，内核线程由 OS 进行管理。Goroutine 更轻量，因此我们可以轻松创建数万 goroutine。一个内核线程可以管理多个 goroutine，当其中一个 goroutine 阻塞时，内核线程可以调度其他的 goroutine 来运行，内核线程本身不会阻塞。这就是通常我们说的 `M:N` 模型：

![m:n模型](https://tva1.sinaimg.cn/large/008i3skNly1gwafpo96f1j30um0b6aap.jpg)

`M:N` 模型通常由三部分构成：M、P、G。M 是内核线程，负责运行 goroutine；P 是 context，保存 goroutine 运行所需要的上下文，它还维护了可运行（runnable）的 goroutine 列表；G 则是待运行的 goroutine。M 和 P 是 G 运行的基础。

![GPM模型](https://tva1.sinaimg.cn/large/008i3skNly1gwafrgfhy6j30s00qumxs.jpg)

继续回到例子。假设我们只有一个 M，当 G1（`go goroutineA(ch)`） 运行到 `val := <- a` 时，它由本来的 running 状态变成了 waiting 状态（调用了 gopark 之后的结果）：

![G1 running](https://tva1.sinaimg.cn/large/008i3skNly1gwafs7d0ktj30uc0u0aas.jpg)

G1 脱离与 M 的关系，但调度器可不会让 M 闲着，所以会接着调度另一个 goroutine 来运行：

![G1 waiting](https://tva1.sinaimg.cn/large/008i3skNly1gwafsxw0s1j30uy0p6aas.jpg)

G2 也是同样的遭遇。现在 G1 和 G2 都被挂起了，等待着一个 sender 往 channel 里发送数据，才能得到解救。

#### 发送

第 17 行向 channel 发送了一个元素 3。发送操作最终转化为 `chansend` 函数

```go
// 位于 src/runtime/chan.go

func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	// 如果 channel 是 nil
	if c == nil {
		// 不能阻塞，直接返回 false，表示未发送成功
		if !block {
			return false
		}
		// 当前 goroutine 被挂起
		gopark(nil, nil, "chan send (nil chan)", traceEvGoStop, 2)
		throw("unreachable")
	}

	// 省略 debug 相关……

	// 对于不阻塞的 send，快速检测失败场景
	//
	// 如果 channel 未关闭且 channel 没有多余的缓冲空间。这可能是：
	// 1. channel 是非缓冲型的，且等待接收队列里没有 goroutine
	// 2. channel 是缓冲型的，但循环数组已经装满了元素
	if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
		(c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
		return false
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	// 锁住 channel，并发安全
	lock(&c.lock)

	// 如果 channel 关闭了
	if c.closed != 0 {
		// 解锁
		unlock(&c.lock)
		// 直接 panic
		panic(plainError("send on closed channel"))
	}

	// 如果接收队列里有 goroutine，直接将要发送的数据拷贝到接收 goroutine
	if sg := c.recvq.dequeue(); sg != nil {
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}
	
  // 对于缓冲型的 channel，如果还有缓冲空间
	if c.qcount < c.dataqsiz {
		// qp 指向 buf 的 sendx 位置
		qp := chanbuf(c, c.sendx)

		// ……

		// 将数据从 ep 处拷贝到 qp
		typedmemmove(c.elemtype, qp, ep)
		// 发送游标值加 1
		c.sendx++
		// 如果发送游标值等于容量值，游标值归 0
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		// 缓冲区的元素数量加一
		c.qcount++

		// 解锁
		unlock(&c.lock)
		return true
	}

	// 如果不需要阻塞，则直接返回错误
	if !block {
		unlock(&c.lock)
		return false
	}

	// channel 满了，发送方会被阻塞。接下来会构造一个 sudog

	// 获取当前 goroutine 的指针
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}

	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.selectdone = nil
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	
	// 当前 goroutine 进入发送等待队列
	c.sendq.enqueue(mysg)

	// 当前 goroutine 被挂起
	goparkunlock(&c.lock, "chan send", traceEvGoBlockSend, 3)

	// 从这里开始被唤醒了（channel 有机会可以发送了）
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	if gp.param == nil {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		// 被唤醒后，channel 关闭了。坑爹啊，panic
		panic(plainError("send on closed channel"))
	}
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	// 去掉 mysg 上绑定的 channel
	mysg.c = nil
	releaseSudog(mysg)
	return true
}
```

上面的代码注释地比较详细了，我们来详细看看。

- 如果检测到 channel 是空的，当前 goroutine 会被挂起。
- 对于不阻塞的发送操作，如果 channel 未关闭并且没有多余的缓冲空间（说明：a. channel 是非缓冲型的，且等待接收队列里没有 goroutine；b. channel 是缓冲型的，但循环数组已经装满了元素）

如果检测到 channel 已经关闭，直接 panic。

如果能从等待接收队列 recvq 里出队一个 sudog（代表一个 goroutine），说明此时 channel 是空的，没有元素，所以才会有等待接收者。这时会调用 send 函数将元素直接从发送者的栈拷贝到接收者的栈，关键操作由 `sendDirect` 函数完成。

```go
// send 函数处理向一个空的 channel 发送操作

// ep 指向被发送的元素，会被直接拷贝到接收的 goroutine
// 之后，接收的 goroutine 会被唤醒
// c 必须是空的（因为等待队列里有 goroutine，肯定是空的）
// c 必须被上锁，发送操作执行完后，会使用 unlockf 函数解锁
// sg 必须已经从等待队列里取出来了
// ep 必须是非空，并且它指向堆或调用者的栈

func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	// 省略一些用不到的
	// ……

	// sg.elem 指向接收到的值存放的位置，如 val <- ch，指的就是 &val
	if sg.elem != nil {
		// 直接拷贝内存（从发送者到接收者）
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	// sudog 上绑定的 goroutine
	gp := sg.g
	// 解锁
	unlockf()
	gp.param = unsafe.Pointer(sg)
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	// 唤醒接收的 goroutine. skip 和打印栈相关，暂时不理会
	goready(gp, skip+1)
}

// 向一个非缓冲型的 channel 发送数据、从一个无元素的（非缓冲型或缓冲型但空）的 channel
// 接收数据，都会导致一个 goroutine 直接操作另一个 goroutine 的栈
// 由于 GC 假设对栈的写操作只能发生在 goroutine 正在运行中并且由当前 goroutine 来写
// 所以这里实际上违反了这个假设。可能会造成一些问题，所以需要用到写屏障来规避
func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
	// src 在当前 goroutine 的栈上，dst 是另一个 goroutine 的栈

	// 直接进行内存"搬迁"
	// 如果目标地址的栈发生了栈收缩，当我们读出了 sg.elem 后
	// 就不能修改真正的 dst 位置的值了
	// 因此需要在读和写之前加上一个屏障
	dst := sg.elem
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
	memmove(dst, src, t.size)
}
```

然后，解锁、唤醒接收者，等待调度器的光临，接收者也得以重见天日，可以继续执行接收操作之后的代码了。

 在发送小节里我们说到 G1 和 G2 现在被挂起来了，等待 sender 的解救。在第 17 行，主协程向 ch 发送了一个元素 3，来看下接下来会发生什么。

根据前面源码分析的结果，我们知道，sender 发现 ch 的 recvq 里有 receiver 在等待着接收，就会出队一个 sudog，把 recvq 里 first 指针的 sudo “推举”出来了，并将其加入到 P 的可运行 goroutine 队列中。

然后，sender 把发送元素拷贝到 sudog 的 elem 地址处，最后会调用 goready 将 G1 唤醒，状态变为 runnable。

![goroutine send](https://tva1.sinaimg.cn/large/008i3skNly1gwag0jy14yj310g0l8754.jpg)

### channel触发panic的三种情况

1. 向一个关闭的 channel 进行写操作(读操作不会触发panic，会返回channel 类型的零值)
2. 关闭一个 nil 的 channel
3. 重复关闭一个 channel。

### 参考文档

- [Go基础系列：channel入门](https://www.cnblogs.com/f-ck-need-u/p/9986335.html)
- [Go Channel 详解](https://colobu.com/2016/04/14/Golang-Channels/) //鸟窝大佬
- [go中通道channel的使用及原理](https://www.cnblogs.com/33debug/p/11889825.html)   // 推荐阅读
- [深度解密Go语言之channel](https://www.qcrao.com/2019/07/22/dive-into-go-channel/)      // 强烈推荐阅读绕全成大佬的所有文章
- [golang runtime源码阅读 channal实现](https://blog.csdn.net/ythunder/article/details/102541212) // 这篇文章也挺不错的

