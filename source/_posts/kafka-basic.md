---
title: kafka 基本原理
date: 2024-03-18 12:46:06
tags:
- kafka
categories:
- kafka
---

### 概述

Kafka是一个分布式数据流平台，可以运行在单台服务器上，也可以在多台服务器上部署形成集群。它提供了发布和订阅功能，使用者可以发送数据到Kafka中，也可以从Kafka中读取数据（以便进行后续的处理）。Kafka具有**高吞吐、低延迟、高容错、可水平扩展、支持流数据处理**等特点。

### Kafka架构概念

Kafka作为一个高度可扩展可容错的消息系统，一个典型的kafka集群中包含若干producer，若干broker，若干consumer，以及一个Zookeeper集群。Kafka通过Zookeeper管理集群配置，选举leader，以及在consumer group发生变化时进行rebalance。producer使用push模式将消息发布到broker，consumer使用pull模式从broker订阅并消费消息：

![kafka架构](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/202403181639076.png)

下面是kafka的一些专业术语：

- Broker：消息中间件处理结点，一个Kafka节点就是一个broker，多个broker可以组成一个Kafka集群。
- Topic：一类消息，Kafka集群能够同时负责多个topic的分发。
- Partition：topic物理上的分组，一个topic可以分为多个partition，每个partition是一个有序的队列。
- Replication：每一个分区都有多个副本，副本的作用是做备胎。当主分区（Leader）故障的时候会选择一个备胎（Follower）上位，成为Leader。**在kafka中默认副本的最大数量是10个，且副本的数量不能大于Broker的数量**，follower和leader绝对是在不同的机器，同一机器对同一个分区也只可能存放一个副本（包括自己）。
- Segment：partition物理上由多个segment组成。
- offset：每个partition都由一系列有序的、不可变的消息组成，这些消息被连续的追加到partition中。partition中的每个消息都有一个连续的序列号叫做offset，用于partition唯一标识一条消息。
- Producer：负责发布消息到Kafka broker。
- Consumer：消息消费者，向Kafka broker读取消息的客户端。
- Consumer Group：每个Consumer属于一个特定的Consumer Group。
- Leader：每个分区多个副本的“主”副本，生产者发送数据的对象，以及消费者消费数据的对象，都是 Leader。
- Follower：每个分区多个副本的“从”副本，实时从 Leader 中同步数据，保持和 Leader 数据的同步。Leader 发生故障时，某个 Follower 还会成为新的 Leader。
- ZooKeeper：Kafka 集群能够正常工作，需要依赖于 ZooKeeper，ZooKeeper 帮助 Kafka 存储和管理集群信息。

Kafka实现了零拷贝原理来快速移动数据，避免了内核之间的切换。Kafka可以将数据记录分批发送，从生产者到文件系统（Kafka主题日志）到消费者，可以端到端的查看这些批次的数据。批处理能够进行更有效的数据压缩并减少I/O延迟，Kafka采取顺序写入磁盘的方式，避免了随机磁盘寻址的浪费，总结一下其实就是四个要点：

- 顺序读写：因为硬盘是机械结构，每次读写都会寻址，写入，其中寻址是一个“机械动作”，它是最耗时的。所以硬盘最“讨厌”随机I/O，最喜欢顺序I/O。为了提高读写硬盘的速度，Kafka就是使用顺序I/O。
- 零拷贝：在Linux Kernal 2.2之后出现了一种叫做“零拷贝(zero-copy)”系统调用机制，就是跳过“用户缓冲区”的拷贝，建立一个磁盘空间和内存空间的直接映射，数据不再复制到“用户态缓冲区”系统上下文切换减少2次，可以提升一倍性能。
- 消息压缩：消息都是经过压缩传递、存储的，降低网络与磁盘的负担。
- 分批发送：批量处理是一种非常有效的提升系统吞吐量的方法，在Kafka内部，消息都是以“批”为单位处理的。

### kafka topic的基本组成

在kafka中，消息都是以 topic 进行分类的，生产者和消费者都是面向topic的，在 kafka 中，一个 topic 可以分为多个 partition，一个 partition可以分为多个segment, 每个 segment 对应两个文件：.index 和 .log 文件；topic 是逻辑上的概念，而 patition 是物理上的概念，每个 patition 对应一个 log 文件，而 log 文件中存储的就是 producer 生产的数据，patition 生产的数据会被不断的添加到 log 文件的末端，且每条数据都有自己的 offset。消费组中的每个消费者，都是实时记录自己消费到哪个 offset，以便出错恢复，从上次的位置继续消费。

![kafka topic组成](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/202403181653360.png)

### 消息存储机制

由于生产者生产的消息会不断追加到 log 文件末尾，为防止 log 文件过大导致数据定位效率低下，Kafka 采取了分片和索引机制，将每个 partition 分为多个 segment。每个 segment 对应两个文件——.index文件和 .log文件。这些文件位于一个文件夹下，该文件夹的命名规则为：topic名称+分区序号。

如下，我们创建一个只有一个分区一个副本的 topic：

```
bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic dawn
```

然后可以在 kafka-logs 目录（server.properties 默认配置）下看到会有个名为 dawn-0 的文件夹。如果，starfish 这个 topic 有三个分区，则其对应的文件夹为 dawn-0，dawn-1，dawn-2。

![消息存储机制](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/202403181706158.png)

这些文件的含义如下：

| 类别                    | 作用                                                         |
| :---------------------- | :----------------------------------------------------------- |
| .index                  | 偏移量索引文件，存储数据对应的偏移量                         |
| .timestamp              | 时间戳索引文件                                               |
| .log                    | 日志文件，存储生产者生产的数据                               |
| .snaphot                | 快照文件                                                     |
| leader-epoch-checkpoint | 保存了每一任leader开始写入消息时的offset，会定时更新。follower被选为leader时会根据这个确定哪些消息可用 |

index 和 log 文件以当前 segment 的第一条消息的 offset 命名。偏移量 offset 是一个 64 位的长整形数，固定是20 位数字，长度未达到，用 0 进行填补，索引文件和日志文件都由此作为文件名命名规则。

从上图可以看出，我们的偏移量是从 0 开始的，.index 和 .log 文件名称都为 00000000000000000000。

在server.properties 文件中会配置日志文件的最大值，当生产者生产数据量较多，一个 segment 存储不下触发分片时，在日志 topic 目录下会看到类似如下所示的文件：

```
00000000000000000000.index 
00000000000000000000.log 
00000000000000170410.index 
00000000000000170410.log 
00000000000000239430.index 
00000000000000239430.log
```

### Topic 副本机制

在Kafka中，Topic的同一个 partition 可能会有多个 replication( 对应 server.properties 配置中的 default.replication.factor=N)。

没有 replication 的情况下，一旦 broker 宕机，其上所有 patition 的数据都不可被消费，同时 producer 也不能再将数据存于其上的 patition。比如一个有 3 台 Broker 的 Kafka 集群上的副本分布情况。主题 1 分区 0 的 3 个副本分散在 3 台 Broker 上，其他主题分区的副本也都散落在不同的 Broker 上，从而实现数据冗余。

![topic副本机制](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/202403181724240.png)

​	引入 replication 之后，同一个 partition 可能会有多个 replication，而这时需要在这些 replication 之间选出一 个 leader， producer 和 consumer 只与这个 leader 交互，其它 replication 作为 follower 从 leader 中复制数据。

![Leader选举机制](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/202403181724105.png)

### kafka Producer写入流程

producer 写入消息流程如下：

1. producer连接ZK集群，从 zookeeper 中拿到对应 topic 的 partition 信息和partition的Leader的相关信息
2. producer 连接leader 对应的broker, 并将消息发送给该 leader
3. leader 将消息写入本地 log
4. followers 从 leader pull 消息，写入本地 log 后向 leader 发送 ACK
5. leader 收到所有 ISR 中的 replication 的 ACK 后，增加 HW(high watermark，最后 commit 的 offset)并向 producer 发送 ACK

![producer写入流程](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/202403181723683.png)

producer 采用推(push) 模式将消息发布到 broker，每条消息都被追加(append) 到分区(patition) 中，属于顺序写磁盘(顺序写磁盘效率比随机写内存要高，保障 kafka 吞吐率)。

#### 分区策略

​		Kafka 的消息组织方式实际上是三级结构：主题 - 分区 - 消息。因此topic下的每条消息只会保存在某一个分区中，而不会在多个分区中被保存多份。分区提供了系统的负载均衡能力，能够实现系统的高伸缩性，不同的分区能够被放置到不同节点的机器上，而数据的读写操作也都是针对分区这个粒度而进行的，这样每个节点的机器都能独立地执行各自分区的读写请求处理。这样，当性能不足的时候可以通过添加新的节点机器来增加整体系统的吞吐量。

​		producer将消息发送到哪个分区又分区策略决定，kafka为我们提供了默认的分区策略，同时支持用户自定义分区策略，常见的分区策略有：

​	**1. 轮询策略：**Round-robin 策略，即顺序分配到每个分区。该策略总是能保证消息最大限度地被平均分配到所有分区上，故默认情况下它是最合理的分区策略，也是最常用的分区策略之一。

![round-robin分区策略](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/202403182042927.png)

**2. 随机策略-Randomness 策略：**随机就是随意地将消息放置到任意一个分区上。

![random策略](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/202403182044021.png)

**3. 按消息键保序策略- Key ordering 策略：**为每条消息定义消息键Key，相同 Key 的所有消息都进入到相同的分区里面。

数据分区投递的基本原则：

1. 如果指定的partition，那么直接进入该partition。
2. 如果没有指定partition，但是指定了key，将 key 的 hash 值与 topic 的 partition 数进行取余得到 partition 值
3. 如果既没有指定partition，也没有指定key，使用轮询策略选择partition。

#### 批量发送

当生产者发送多个消息到同一个topic时，为了减少网络带来的开销，kafka会对消息进行批量发送；主要涉及3个参数如下，满足其一即会批量发送：

- batch-size ：超过收集的消息数量的最大量，默认16KB
- buffer-memory ：超过收集的消息占用的最大内存 , 默认32M
- linger.ms ：超过收集的时间的最大等待时长，单位：毫秒，默认为0ms, 即有消息就发送

#### 数据可靠性保证

为保证 producer 发送的数据，能可靠的发送到指定的 topic，topic 的每个 partition 收到 producer 数据后，都需要向 producer 发送 ack ( acknowledgement 确认收到)，如果 producer 收到 ack，就会进行下一轮的发送，否则重新发送数据。

![数据可靠性保证](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/202403182050435.png)

##### 数据同步策略

Leader将消息写入到本地后，需要确保有 Follower 与 Leader 同步完成后才能发送 ACK，这样才能保证 Leader 挂掉之后，能在 Follower 中选举出新的 Leader 而不丢数据，Follower数据同步发送ACK主要有两种策略：

| 方案                        | 优点                                                   | 缺点                                                  |
| :-------------------------- | :----------------------------------------------------- | :---------------------------------------------------- |
| 半数以上完成同步，就发送ack | 延迟低                                                 | 选举新的 leader 时，容忍n台节点的故障，需要2n+1个副本 |
| 全部完成同步，才发送ack     | 选举新的 leader 时，容忍n台节点的故障，需要 n+1 个副本 | 延迟高                                                |

Kafka 选择了第二种方案，原因如下：

- 同样为了容忍 n 台节点的故障，第一种方案需要的副本数相对较多，而 Kafka 的每个分区都有大量的数据，第一种方案会造成大量的数据冗余;
- 虽然第二种方案的网络延迟会比较高，但网络延迟对 Kafka 的影响较小。

##### LSR

采用第二种方案之后，会出现一个问题：leader 收到数据，所有 follower 都开始同步数据，但此时有一个 follower 故障导致不能与 leader 保持同步，此时leader 得一直等下去，直到它完成同步，才能发送 ack，这会造成消息无法确定。

为了解决此问题，leader 维护了一个动态的 in-sync replica set(ISR)，意为和 leader 保持同步的 follower 集合。当 ISR 中的follower 完成数据的同步之后，leader 就会给 follower 发送 ack。如果 follower 长时间未向 leader 同步数据，则该 follower 将会被踢出 ISR，该时间阈值由 replica.lag.time.max.ms 参数设定。leader 发生故障之后，就会从 ISR 中选举新的 leader。

##### ack应答机制

对于某些不太重要的数据，对数据的可靠性要求不是很高，能够容忍数据的少量丢失，所以没必要等 ISR 中的follower全部接收成功。所以Kafka为用户提供了三种可靠性级别，用户根据对可靠性和延迟的要求进行权衡

**0：**producer 不等待 broker 的 ack，这一操作提供了一个最低的延迟，broker 一接收到还没有写入磁盘就已经返回，当 broker 故障时有可能丢失数据;

**1：**producer 等待 broker 的 ack，partition 的 leader 落盘成功后返回 ack，如果在 follower 同步成功之前 leader 故障，那么将会丢失数据;

**-1(all)：**producer 等待 broker 的 ack，partition 的 leader 和 follower 全部落盘成功后才返回 ack。但是 如果在 follower 同步完成后，broker 发送 ack 之前，leader 发生故障，那么就会造成数据重复。

![ack应答机制](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/202403182108033.png)

##### 故障处理

​	由于我们并不能保证 Kafka 集群中每时每刻 follower 的长度都和 leader 一致(即数据同步是有时延的)，那么当leader 挂掉选举某个 follower 为新的 leader 的时候(原先挂掉的 leader 恢复了成为了 follower)，可能会出现leader 的数据比 follower 还少的情况。为了解决这种数据量不一致带来的混乱情况，Kafka 提出了以下概念：

- LEO(Log End Offset)：指的是每个副本最后一个offset;
- HW(High Wather)：指的是消费者能见到的最大的 offset，ISR 队列中最小的 LEO。

![LEO & HW](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/202403182111530.png)

**消费者和 leader 通信时，只能消费 HW 之前的数据，HW 之后的数据对消费者不可见**，因此：

- 当follower发生故障时：follower 发生故障后会被临时踢出 ISR，待该 follower 恢复后，follower 会读取本地磁盘记录的上次的 HW，并将 log 文件高于 HW 的部分截取掉，从 HW 开始向 leader 进行同步。等该 follower 的 LEO 大于等于该 Partition 的 HW，即 follower 追上 leader 之后，就可以重新加入 ISR 了。
- 当leader发生故障时：leader 发生故障之后，会从 ISR 中选出一个新的 leader，之后，为保证多个副本之间的数据一致性，其余的 follower 会先将各自的 log 文件高于 HW 的部分截掉，然后从新的 leader 同步数据。

所以数据一致性并不能保证数据不丢失或者不重复，这是由 ack 控制的。HW 规则只能保证副本之间的数据一致性!

#### Exactly Once

将服务器的 ACK 级别设置为 -1，可以保证 Producer 到 Server 之间不会丢失数据，即 At Least Once 语义。相对的，将服务器 ACK 级别设置为 0，可以保证生产者每条消息只会被发送一次，即 At Most Once语义。

At Least Once 可以保证数据不丢失，但是不能保证数据不重复。相对的，At Most Once 可以保证数据不重复，但是不能保证数据不丢失。但是，对于一些非常重要的信息，比如说交易数据，下游数据消费者要求数据既不重复也不丢失，即 Exactly Once 语义。在 0.11 版本以前的 Kafka，对此是无能为力的，只能保证数据不丢失，再在下游消费者对数据做全局去重。对于多个下游应用的情况，每个都需要单独做全局去重，这就对性能造成了很大的影响。

0.11 版本的 Kafka，引入了一项重大特性：幂等性。所谓的幂等性就是指 Producer 不论向 Server 发送多少次重复数据。Server 端都会只持久化一条，幂等性结合 At Least Once 语义，就构成了 Kafka 的 Exactily Once 语义，即：At Least Once + 幂等性 = Exactly Once

要启用幂等性，只需要将 Producer 的参数中 enable.idompotence 设置为 true 即可。Kafka 的幂等性实现其实就是将原来下游需要做的去重放在了数据上游。开启幂等性的 Producer 在初始化的时候会被分配一个 PID，发往同一 Partition 的消息会附带 Sequence Number。而 Broker 端会对Sequence Number做校验，如果发现PID对应的Sequence Number重复，就会丢弃此消息

但是 PID 重启就会变化，同时不同的 Partition 也具有不同主键，所以幂等性无法保证跨分区会话的 Exactly Once。

### Kafka Consumer消费过程

Consumer 采用 Pull（拉取）模式从 Broker 中读取数据。Pull 模式则可以根据 Consumer 的消费能力以适当的速率消费消息。Pull 模式不足之处是，如果 Kafka没有数据，消费者可能会陷入循环中，一直返回空数据。

因为消费者从 Broker 主动拉取数据，需要维护一个长轮询，针对这一点， Kafka 的消费者在消费数据时会传入一个时长参数 timeout。如果当前没有数据可供消费，Consumer 会等待一段时间之后再返回，这段时长即为 timeout。

#### 消费者组

消费者是以 consumer group 消费者组的方式工作，由一个或者多个消费者组成一个组， 共同消费一个 topic。每个分区在同一时间只能由 group 中的一个消费者读取，但是多个 group 可以同时消费这个 partition。

通过消费者组的模式，消费者可以通过水平扩展的方式同时读取大量的消息。另外，如果一个消费者失败了，那么其他的 group 成员会自动负载均衡读取之前失败的消费者读取的分区。

消费者组最为重要的一个功能是实现广播与单播的功能。一个消费者组可以确保其所订阅的 Topic 的每个分区只能被从属于该消费者组中的唯一一个消费者所消费；如果不同的消费者组订阅了同一个 Topic，那么这些消费者组之间是彼此独立的，不会受到相互的干扰。

如果我们希望一条消息可以被多个消费者所消费，那么可以将这些消费者放到不同的消费者组中，这实际上就是广播的效果；如果希望一条消息只能被一个消费者所消费，那么可以将这些消费者放到同一个消费者组中，这实际上就是单播的效果。

![consumer group](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/202403182133768.png)

#### 分区策略

一个 consumer group 中有多个 consumer，一个 topic 有多个 partition，所以必然会涉及到 partition 的分配问题，即确定哪个 partition 由哪个 consumer 来消费。

Kafka 有两种分配策略，一是 RoundRobin，一是 Range。

**1. RoundRobin：**RoundRobin 即轮询的意思，比如现在有一个三个消费者 ConsumerA、ConsumerB 和 ConsumerC 组成的消费者组，同时消费 TopicA 主题消息，TopicA 分为 7 个分区，如果采用 RoundRobin 分配策略，过程如下所示：

![RoundRobin](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/202403182137474.png)

**2. Range：** Range 顾名思义就是按范围划分的意思，Kafka 默认采用 Range 分配策略，

比如现在有一个三个消费者 ConsumerA、ConsumerB 和 ConsumerC 组成的消费者组，同时消费 TopicA 主题消息，TopicA分为7个分区，如果采用 Range 分配策略，过程如下所示：

![range](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/202403182138001.png)

#### offset的维护

由于 consumer 在消费过程中可能会出现断电宕机等故障，consumer 恢复后，需要从故障前的位置继续消费，所以 consumer 需要实时记录自己消费到了哪个 offset，以便故障恢复后继续消费。

Kafka 0.9 版本之前，consumer 默认将 offset 保存在 Zookeeper 中，从 0.9 版本开始，consumer 默认将 offset保存在 Kafka 一个内置的 topic 中，该 topic 为 _consumer_offsets。

### kafka事务

Kafka 事务基于幂等性实现，**通过事务机制，Kafka 可以实现对多个 Topic 、多个 Partition 的原子性的写入**，即处于同一个事务内的所有消息，最终结果是要么全部写成功，要么全部写失败。

> Kafka 事务分为生产者事务和消费者事务，但它们并不是强绑定的关系，消费者主要依赖自身对事务进行控制，因此这里我们主要讨论的是生产者事务。

为了了实现跨分区跨会话的事务，需要引入一个全局唯一的 TransactionID，并将 Producer 获得的 PID 和Transaction ID 绑定。这样当 Producer 重启后就可以通过正在进行的 TransactionID 获得原来的 PID。

为了管理 Transaction，Kafka 引入了一个新的组件 Transaction Coordinator。Producer 就是通过和 Transaction Coordinator 交互获得 Transaction ID 对应的任务状态。Transaction Coordinator 还负责将事务所有写入 Kafka 的一个内部 Topic，这样即使整个服务重启，由于事务状态得到保存，进行中的事务状态可以得到恢复，从而继续进行。

![kafka事务](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/202403182145718.png)

### 参考文档

- [一文讲清 Kafka 工作流程和存储机制](https://www.51cto.com/article/622050.html) --- 推荐阅读
- [一文理解Kafka的设计原理](https://zhuanlan.zhihu.com/p/610324541)  --- 推荐阅读
- [Kafka 介紹 + Golang程序实现](https://ftn8205.medium.com/kafka-%E4%BB%8B%E7%B4%B9-golang%E7%A8%8B%E5%BC%8F%E5%AF%A6%E4%BD%9C-2b108481369e)  --- 简单看看
- [kafka序列文章](https://yangyangmm.cn/tags/Kafka/page/2/#board) --- 简单看看

