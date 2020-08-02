---
title: linux网卡收发包流程
date: 2020-07-09 14:55:33
tags:
- linux
categories:
- linux
---

### 网卡数据包接收流程

1. 网卡收到数据包。
2. 将数据包从网卡硬件缓存转移到服务器内存中。
3. 通过中断请求，通知内核处理。
4. 经过 TCP/IP 协议逐层处理。
5. 应用程序通过 read() 从 socket buffer 读取数据。

![网卡收包流程](./linux-nic/nic.png)

### 网卡收到的数据包转移到主机内存（NIC 与驱动交互）

NIC(网卡)在接收到数据包之后，首先需要将数据同步到内核中，这中间的桥梁是 `rx ring buffer`。它是由 NIC 和驱动程序共享的一片区域，事实上，`rx ring buffer`存储的并不是实际的 packet 数据，而是一个描述符，这个描述符指向了它真正的存储地址，具体流程如下：

1. 驱动在内存中分配一片缓冲区用来接收数据包，叫做 sk_buffer；
2. 将上述缓冲区的地址和大小（即接收描述符），加入到 rx ring buffer。描述符中的缓冲区地址是 DMA 使用的物理地址；
3. 驱动通知网卡有一个新的描述符；
4. 网卡从 rx ring buffer 中取出描述符，从而获知缓冲区的地址和大小；
5. 网卡收到新的数据包；
6. 网卡将新数据包通过 DMA 直接写到 sk_buffer 中。

![Nic与驱动交互](./linux-nic/nic2.png)

当驱动处理速度跟不上网卡收包速度时，驱动来不及分配缓冲区，NIC 接收到的数据包无法及时写到 sk_buffer，就会产生堆积，当 NIC 内部缓冲区写满后，就会丢弃部分数据，引起丢包。这部分丢包为 rx_fifo_errors，在 /proc/net/dev 中体现为 fifo 字段增长，在 ifconfig 中体现为 overruns 指标增长。

### 系统内核处理（驱动与 Linux 内核交互）

这个时候，数据包已经被转移到了 sk_buffer 中。前文提到，这是驱动程序在内存中分配的一片缓冲区，并且是通过 DMA 写入的，这种方式不依赖 CPU 直接将数据写到了内存中，意味着对内核来说，其实并不知道已经有新数据到了内存中。那么如何让内核知道有新数据进来了呢？答案就是中断，通过中断告诉内核有新数据进来了，并需要进行后续处理。

提到中断，就涉及到硬中断和软中断，首先需要简单了解一下它们的区别：

- 硬中断：由硬件自己生成，具有随机性，硬中断被 CPU 接收后，触发执行中断处理程序。中断处理程序只会处理关键性的、短时间内可以处理完的工作，剩余耗时较长工作，会放到中断之后，由软中断来完成。硬中断也被称为上半部分。
- 软中断：由硬中断对应的中断处理程序生成，往往是预先在代码里实现好的，不具有随机性。（除此之外，也有应用程序触发的软中断，与本文讨论的网卡收包无关。）也被称为下半部分。
当 NIC 把数据包通过 DMA 复制到内核缓冲区 sk_buffer 后，NIC 立即发起一个硬件中断。CPU 接收后，首先进入上半部分，网卡中断对应的中断处理程序是网卡驱动程序的一部分，之后由它发起软中断，进入下半部分，开始消费 sk_buffer 中的数据，交给内核协议栈处理。

 ![系统内核处理](./linux-nic/nic1.png)

通过中断，能够快速及时地响应网卡数据请求，但如果数据量大，那么会产生大量中断请求，CPU 大部分时间都忙于处理中断，效率很低。为了解决这个问题，现在的内核及驱动都采用一种叫 NAPI（new API）的方式进行数据处理，其原理可以简单理解为 中断 + 轮询，在数据量大时，一次中断后通过轮询接收一定数量包再返回，避免产生多次中断。


### Ring Buffer
Ring Buffer 相关的收消息过程大致如下：
1. NIC (network interface card) 在系统启动过程中会向系统注册自己的各种信息，系统会分配 Ring Buffer 队列也会分配一块专门的内核内存区域给 NIC 用于存放传输上来的数据包。struct sk_buff 是专门存放各种网络传输数据包的内存接口，在收到数据存放到 NIC 专用内核内存区域后，[http://elixir.free-electrons.com/linux/v4.4/source/include/linux/skbuff.h#L706](sk_buff 内有个 data 指针会指向这块内存)

2. Ring Buffer 队列内存放的是一个个 Packet Descriptor ，其有两种状态： ready 和 used 。初始时 Descriptor 是空的，指向一个空的 sk_buff，处在 ready 状态。当有数据时，DMA 负责从 NIC 取数据，并在 Ring Buffer 上按顺序找到下一个 ready 的 Descriptor，将数据存入该 Descriptor 指向的 sk_buff 中，并标记槽为 used。因为是按顺序找 ready 的槽，所以 Ring Buffer 是个 FIFO 的队列。

3. 当 DMA 读完数据之后，NIC 会触发一个 IRQ(中断请求) 让 CPU 去处理收到的数据。因为每次触发 IRQ(中断请求) 后 CPU 都要花费时间去处理 Interrupt Handler，如果 NIC 每收到一个 Packet 都触发一个 IRQ 会导致 CPU 花费大量的时间在处理 Interrupt Handler，处理完后又只能从 Ring Buffer 中拿出一个 Packet，虽然 Interrupt Handler 执行时间很短，但这么做也非常低效，并会给 CPU 带去很多负担。所以目前都是采用一个叫做 New API(NAPI) 的机制，去对 IRQ 做合并以减少 IRQ 次数。

![Ring Buffer](https://ylgrgyq.github.io/2017/07/23/linux-receive-packet-1/ring-buffer.png)

### NAPI
NAPI 主要是让 NIC 的 driver 能注册一个 poll 函数，之后 NAPI 的 subsystem 能通过 poll 函数去从 Ring Buffer 中批量拉取收到的数据。主要事件及其顺序如下：
1. NIC driver 初始化时向 Kernel 注册 poll 函数，用于后续从 Ring Buffer 拉取收到的数据
2. driver 注册开启 NAPI，这个机制默认是关闭的，只有支持 NAPI 的 driver 才会去开启
3. 收到数据后 NIC 通过 DMA 将数据存到内存
4. NIC 触发一个 IRQ，并触发 CPU 开始执行 driver 注册的 Interrupt Handler
5. driver 的 Interrupt Handler 通过 napi_schedule 函数触发 softirq (NET_RX_SOFTIRQ) 来唤醒 NAPI subsystem，NET_RX_SOFTIRQ 的 handler 是 net_rx_action 会在另一个线程中被执行，在其中会调用 driver 注册的 poll 函数获取收到的 Packet
6. driver 会禁用当前 NIC 的 IRQ，从而能在 poll 完所有数据之前不会再有新的 IRQ
7. 当所有事情做完之后，NAPI subsystem 会被禁用，并且会重新启用 NIC 的 IRQ
8. 回到第三步

从上面的描述可以看出来还缺一些东西，Ring Buffer 上的数据被 poll 走之后是怎么交付上层网络栈继续处理的呢？以及被消耗掉的 sk_buff 是怎么被重新分配重新放入 Ring Buffer 的呢？

这两个工作都在 poll 中完成，上面说过 poll 是个 driver 实现的函数，所以每个 driver 实现可能都不相同。但 poll 的工作基本是一致的就是：

1. 从 Ring Buffer 中将收到的 sk_buff 读取出来
2. 对 sk_buff 做一些基本检查，可能会涉及到将几个 sk_buff 合并因为可能同一个 Frame 被分散放在多个 sk_buff 中
3. 将 sk_buff 交付上层网络栈处理
4. 清理 sk_buff，清理 Ring Buffer 上的 Descriptor 将其指向新分配的 sk_buff 并将状态设置为 ready
5. 更新一些统计数据，比如收到了多少 packet，一共多少字节等

### 软中断处理
1. 遍历 ptype_base 链表，找出 Protocol Layer 中能处理当前数据包的 packet_type 来接着处理数据。所有能处理链路层数据包的协议都会注册到 ptype_base 中。
2. 基本就是在做各种检查以及为 transport 层做一些数据准备。最后如果各种检查都能过，就执行 NF_HOOK。如果有检查不过需要丢弃数据包就会返回 NET_RX_DROP 并在之后会对丢数据包这个事情进行计数。
3. NF_HOOK 比较神，它实际是 HOOK 到一个叫做 Netfilter 的东西，在这里你可以根据各种规则对数据包做过滤以及对数据包做一些修改。如果 HOOK 执行后返回 1 表示 Netfilter 允许继续处理该数据包，就会进入 ip_rcv_finish，HOOK 没有返回 1 则会返回 Netfilter 的结果，数据包不会继续被处理。
4. ip_rcv_finish 负责为 sk_buff 从 IP Route System 中找到路由目标，如果是路由到本机则在下一个处理这个 sk_buff 的协议内(比如上层的 TCP/UDP 协议)还需要从 sk_buff 中找到对应的 socket。

### 参考文档
- https://mp.weixin.qq.com/s/0TGH6zZP_psKyZl4JA-3Qw
- https://ylgrgyq.github.io/2017/07/23/linux-receive-packet-1/
- https://elixir.bootlin.com/linux/v4.4/source/drivers/net/ethernet/intel/igb/igb_main.c#L6361