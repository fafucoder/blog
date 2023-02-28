---
title: RFC QUIC草案文档阅读整理
date: 2022-08-13 22:02:18
tags:
- linux
categories:
- linux
---

### 概述

quic草案的阅读总是晦涩难懂的，如果没有多读几遍，压根就不懂这说的是啥意思，阅读草案之前建议读者先了解相关知识，带着相关知识去阅读更易理解。

### QUIC之基础概念



### QUIC之地址校验

地址校验主要是用于确保端点不会被用于流量放大攻击（traffic amplification attack）。攻击者如果伪造数据包的源地址为受害者的地址，发送大量的数据包给服务端，如果服务端没有进行地址验证，直接响应大量数据包给源地址（受害者），就会被攻击者利用、进行流量放大攻击。

QUIC 针对放大攻击的主要防御措施是验证端点是否能够在其声明的传输地址接收数据包。地址验证在连接建立（connection establishment）期间和连接迁移（connection migration）期间进行。

#### 连接建立时的地址校验

连接建立时，为了验证客户端的地址是否是攻击者伪造的，服务端会生成一个令牌（token）并通过重试包（Retry packet）响应给客户端。客户端需要在后续的初始包（Initial packet）带上这个令牌，以便服务端进行地址验证。

服务端可以在当前连接中通过 NEW_TOKEN 帧预先发布令牌，以便客户端在后续的新连接使用，这是 QUIC 实现 0-RTT 很重要的一个功能。

重试数据包（Retry packet）中提供的令牌只能立即使用，不能用于后续连接的地址验证。而 NEW_TOKEN 帧生成的令牌可以在一个时间范围内使用，这个令牌应该有一个过期时间，可以是显式的过期时间，也可以是可用于动态计算过期时间的时间戳（timestamp）。服务端可以存储过期时间，也可以在令牌中以加密的形式包含它。

![image-20230213235255656](/Users/dawn/Library/Application Support/typora-user-images/image-20230213235255656.png)

**需要注意的是：**

1. 在验证客户端的地址之前，服务端发送的字节数不能超过它接收到的字节数的三倍，用于避免攻击者在地址验证之前进行放大攻击。

2. 客户端必须确保初始数据包（Initial packets）的大小至少1200 字节，如果少于 1200 字节则可以添加 PADDING 帧填充。

3. 如果客户端没有收到来自服务端发送的初始数据包或者握手包，而且客户端没有发送额外的初始包或者握手包，那么这会引发死锁。因此为了防止死锁，客户端必须在探测超时(PTO)时（重新）发送数据包，如果客户端没有握手秘钥，那么他必须（重新）发送初始化数据包，如果有握手秘钥，那么它应该发送一个握手数据包

#### 路径验证

路径验证用于（端点）连接迁移时校验更新后路径是否可达，在路径验证中，端点会校验本地地址和对端地址间的可达性(这里说的地址是指IP+端口组成的二元组)

路径验证测试是在一条路径上发送给对端的数据包有没有被对端收到，使用地址验证用于确保端点从迁移方收到的数据包不携带伪造的源地址。

##### 启动路径验证

端点通过发送一个PATH_CHALLENGE帧用于启动路径验证，PATH_ChALLENGE帧中的必须包含一个不可预测的 payload，以便它可以将对端响应的PATH_CHALLENGE帧关联起来。

端点可以发送多个 PATH_CHALLENGE 帧以防止数据包丢失。但是不应该在同一个数据包（packet）中发送多个 PATH_CHALLENGE 帧，而是要分别在不同的数据包（packet）中发送。

端点必须将包含PATH_CHALLENGE帧的数据包扩展到至少1200字节(如果不足1200字节需要使用PADDING帧填充)

端点不应该以高于初始化数据包(Initial包)的频率发送PATH_CHALLENGE帧，以确保连接迁移不会比建立新连接带来更多的载荷

##### 响应路径验证

端点在接收到 PATH_CHALLENGE 帧时，必须通过 PATH_RESPONSE 帧响应，PATH_RESPONSE帧的payload跟PATH_CHALLENGE帧一致。除非受到拥塞控制（congestion control）的限制，否则端点不得延迟传输包含 PATH_RESPONSE 帧的数据包。

PATH_RESPONSE 帧必须在接收到 PATH_CHALLENGE 的那条路径上发送。

端点(也)必须将包含 PATH_RESPONSE 帧的数据报扩展到 至少1200 字节。

端点不能发送多个 PATH_RESPONSE 帧来响应一个 PATH_CHALLENGE 帧。

### QUIC之版本协商

QUIC标准化的过程中，发布了多个版本的草案，市面上的QUIC协议实现可能基于不同的版本(daft-29,daft-30),这意味着客户端跟服务端支持的quic协议版本不一样，因此在建立连接时需要先进行版本协商，使用双发都支持的一个版本。

#### 发送版本协商包

客户端发送的第一个包(Initial包)决定服务端是否发送版本协商包，客户端和服务端创建连接时，客户端在首次发起请求时需要带上它支持的协议版本号。

- 如果服务端可以支持客户端的版本， 服务端将为连接的整个生命周期使用这个协议版本。
- 如果服务端不支持该版本，服务端将发送版本协商包附上它所支持的版本集合，这将增加 1-RTT（Round-Trip Time） 的延迟开销。

需要注意的是：

- 为了减少放大攻击（amplification attacks），QUIC 协议要求客户端发送的初始数据包大小（Initial Datagram Size）最少为 1200 字节。如果初始数据包小于 1200 字节，需要使用 PADDING frame 填充，不然该数据包会被服务端丢弃。
- 只有服务端可以发送版本协商包（Version Negotiation packet），客户端不能发送。
- 服务端识别到 0-RTT 数据包（之前有成功连接过），可以选择不发送版本协商包，以减少额外的 1-RTT 版本协商延迟。
- 服务端响应发送的初始（Initial）数据包或版本协商数据包可能丢失，客户端可以继续发送新的数据包、直到它成功接收到服务端响应，或者放弃连接尝试。

![image-20230213235419054](/Users/dawn/Library/Application Support/typora-user-images/image-20230213235419054.png)

#### 处理版本协商包

客户端收到版本协商包后，从服务端所支持的版本集合里面挑选它所支持的版本。

- 如果所有的版本都不支持，则客户端需要丢弃连接。
- 如果有匹配到支持的版本，客户端尝试使用该版本创建新连接。新连接必须使用一个新的随机目标连接ID（Destination Connection ID）。

如果客户端已接收并成功处理了任何其他包（包括早期的版本协商包），则客户端必须丢弃它后来新收到的版本协商包。

**关于QUIC的版本：**

QUIC 版本使用 32 位无符号数字标识，版本号 0x00000000 被保留用来表示版本协商。版本号 0x00000001 作为 RFC 发布的协议版本

0x?a?a?a?a 格式的版本号被保留（reserved）用于强制执行版本协商

#### 版本协商包格式

![image-20230213235514550](/Users/dawn/Library/Application Support/typora-user-images/image-20230213235514550.png)

注意事项：

- 版本协商包不需要 ACK
- 版本协商包没有 Packet Number 和 Length 字段。因此，它将使用整个 UDP 数据报（datagram）。
- 服务端不能在单个 UDP 数据报（datagram）里面发送多个版本协商包。

### QUIC之连接建立

#### Packet Number及其上下文

Packet Number 为整型变量，其值在 0 到 2^62-1 之间，它也用于生成数据包加密所需的 nonce。通讯双方维护各自的 Packet Number 体系， 并且分为三个独立的上下文空间:

- Initial 空间：所有的 Initial 数据包的 Packet Number 均在这个上下文空间里；
- Handshake 空间：所有的握手数据包；
- 应用数据空间：所有的 0-RTT 和 1-RTT 包。

所谓的 Packet Number 空间，指得是一种上下文关系，在这个上下文关系里，数据包被处理, 被确认。数据包在不同的Packet Number Namespace有不同的加密等级，初始数据包只能使用初始数据包专用的密钥，也只能确认初始数据包。类似的， 握手包只能使用握手包专用的密钥，也只能确认握手数据包。

从 Initial 阶段进入 Handshake 阶段后， Initial 阶段使用的密钥就可以被丢弃了，0-RTT 和 1-RTT 共享同一个 Packet Number 空间，这样做是为了更容易实现这两类数据包的丢包处理算法。

在同一连接同一个 Packet Number 空间里，你不能复用包号，包号必须是单调递增的，当然，具体实现的时候草案并不强制要求每次都递增1， 你可以递增 20，30。当 Packet Number 达到 2^62 -1 时，发送方必须关闭该连接。

需要注意的是：在特定的包号空间里，有些帧是被禁止使用的(https://autumnquiche.github.io/RFC9000_Chinese_Translation/#12.4_Frames_and_Frame_Types)

- Padding帧、Ping帧和Crypto帧*可以*出现在任何数据包号空间中。
- 标志着QUIC层错误（类型为`0x1c`）的连接关闭帧*可以*出现在任何数据包号空间中。标志着应用错误（类型为`0x1d`）的连接关闭帧*必须*只能出现在应用数据空间中。
- ACK帧*可以*出现在任何数据包号空间中，但是只能确认在同一个数据包号空间中的数据包。0-RTT数据包不能包含ACK帧。
- 所有其他类型的帧*必须*只能出现在应用数据空间中。

#### 连接之建立过程

![image-20230213235705155](/Users/dawn/Library/Application Support/typora-user-images/image-20230213235705155.png)

#### 传输参数定义

在连接建立期间，双端会对各自的传输参数作出验证声明，传输参数通过在quic_transport_parameters字段中定义，传输参数包括最大超时时间(max_idle_timeout)， 无状态重置令牌(stateless_reset_token), 最大udp有效载荷（max_udp_payload_size），初始最大数据量(initial_max_data)等,具体参数参考(https://autumnquiche.github.io/RFC9000_Chinese_Translation/#18_Transport_Parameter_Encoding)

![image-20230213235810360](/Users/dawn/Library/Application Support/typora-user-images/image-20230213235810360.png)

同一个传输参数在特定的传输扩展中不能声明多次，一旦握手完成，由对端声明的传输参数就生效了

开启0-RTT，终端需要保存服务端传输参数的值及在连接上收到的任何会话票据（session ticket），终端也需要保存其他任何应用协议或加密握手所需要的信息。但是需要注意的是不是所有的传输参数都需要保存(因为部分参数不会在0-RTT建连期间起作用)， ack_delay_exponent、max_ack_delay、initial_source_connection_id等这些个参数的值不能被保存。

如果终端不支持某一些传输参数，那么必须忽略他(例如B定义了一个传输参数x, A不支持此参数，那么需要忽略他)

#### 协商连接ID

当客户端发送初始包时，会生成不可预测值填充发送的初始包的目标连接ID字段。目标连接ID的长度必须至少8字节。客户端在一条连接上必须使用同一个目标连接ID，直到收到服务端发来的数据包为止。

客户端发送的首个初始数据包的目标连接ID字段用于确定初始数据包的包保护密钥。这些密钥在收到重试数据包后变更。(初始秘钥值是通过对值为0x38762cf7f55934b34d179ae6a4c80cadccbb7f0a的盐和值为目标连接ID进行HKDF算法而得出）

在首次收到从服务端发来的初始数据包或重试数据包后，客户端将服务的提供的源连接ID作为后续发送数据包的目标连接ID，包括任何0-RTT包。这意味着客户端可能需要在连接建立阶段将目标连接ID字段的值变更两次：一次是响应服务端发来的重试数据包，一次是响应服务端发来的初始数据包



### QUIC之连接迁移

### QUIC之连接关闭

### QUIC之不可靠传输

QUIC传输协议[RFC9000]为传输可靠的应用数据流提供了一个安全的、多路复用的连接。QUIC使用携带了多种类型帧的数据包传输数据，需要可靠传输的应用数据流使用 STREAM 帧发送。但是有些应用，尤其是需要传输实时数据的应用，更倾向于使用不可靠传输，QUIC为了实现不可靠传输，通过定义新的Datagram帧类型。

#### Datagram帧传输参数

1. QUIC在握手期间可以用传输参数(name=max_datagram_frame_size，value=0x20)来通告对端是否支持datagram帧，默认值为0表示不支持DATAGRAM帧，大于 0 的值表示端点支持 DATAGRAM 帧类型并且告诉对端自己可以接收datagram帧的最大长度。

2. 端点在握手期间（如果使用0-RTT，则是上一次握手期间），在未收到具有非零值的 max_datagram_frame_size 传输参数之前，不得发送 DATAGRAM 帧， 端点不得发送大于对端通告的 max_datagram_frame_size 长度的 DATAGRAM 帧，如果未收到是否支持datagram帧的通道而收到datagram帧，那么需要以PROTOCOL_VIOLATION类型错误而终止。
3. max_datagram_frame_size 传输参数可以是单向的，也就是可以单端使用。

![image-20230213232140765](/Users/dawn/Library/Application Support/typora-user-images/image-20230213232140765.png)

#### Datagram帧的发送和接收处理

1. 当应用在QUIC连接上发送数据报时，QUIC将生成一个新的 DATAGRAM 帧并在第一个可用数据包中发送。该帧应该尽快投递并且可以与其他帧合并。当 QUIC 端点接收到一个有效的 DATAGRAM 帧时，应该立即传递给应用。

2. 与 STREAM 帧一样，DATAGRAM 帧包含应用数据，并且必须使用 0-RTT 或 1-RTT 密钥进行保护。

3. 虽然 DATAGRAM 帧在丢包检测时不会重传，但它们也是 ACK 触发帧，所以接收方应该支持延迟发送 ACK 帧以对接收到仅包含 DATAGRAM 帧的数据包做出响应，因为即使这些包短期内未被确认，发送方也不会采取任何行动。

4. 与任何 ACK 触发帧一样，当发送方怀疑仅包含 DATAGRAM 帧的数据包丢失时，它会发送探测包以引发更快的 ACK 确认。如果发送方检测到包含特定 DATAGRAM 帧的数据包可能已经丢失，则QUIC实现可以通知应用它认为数据报已丢失了。

5. 如果包含 DATAGRAM 帧的数据包被确认，则QUIC实现可以通知发送方，应用数据报已被成功发送和接收，需要注意的是，对 DATAGRAM 帧的确认仅表明接收方的传输层接收并处理了该帧，并不保证接收方的应用层成功处理了该数据。

6. DATAGRAM帧没有明确的流控信号且DATAGRAM帧不影响其他流的流量控制(也就是DATAGRAM帧不在stream的流量控制范围内)。

7. DATAGRAM 帧的拥塞控制属于连接级别，QUIC实现可以选择让应用指定一个发送过期时间，超过该时间，受拥塞控制的 DATAGRAM 帧应该被丢弃不传输。

### MPQUIC实现多路径传输

#### 多路径协商

1. 端点在握手期间需要使用传输参数(enable_multipath（实验使用0xbabf))来协商是否支持多路径，0表示禁用多路径。
2. 如果任何一个端点，该参数不存在或设置为 0，则端点必须回退到QUIC模式

![image-20230213233123457](/Users/dawn/Library/Application Support/typora-user-images/image-20230213233123457.png)

#### 路径的启动和关闭

1. 当协商多路径选项时，想要使用附加路径的客户端必须首先使用PATH_CHALLENGE 和 PATH_RESPONSE 帧启动地址验证过程。在新路径上接收到来自客户端的数据包后，如果服务器决定使用新路径，则服务器必须执行路径验证，除非它之前已经验证了该地址。

2. 如果验证成功，客户端可以在新路径上发送非探测的 1-RTT 数据包。服务端收到新路径上的非探测1-RTT数据包表示新路径可以使用，但是不能表示连接迁移到新路径。

3. 每个端点可以管理一组路径，如果端点想要关闭指定路径，应该通过PATH_ABANDON帧来终止路径，一旦路径被标记为“已放弃”，端点可以释放与该路径相关的资源，例如已使用的连接ID。

4. 当包含 PATH_ABANDON 帧的数据包对端被确认时，发送 PATH_ABANDON 帧的端点应该认为一条路径已被放弃。当释放该路径的资源时，端点应该为路径上使用的连接 ID 发送一个 RETIRE_CONNECTION_ID 帧，如果有的话。

5. PATH_ABANDON 帧的接收者不应立即释放其资源，而应等待接收到已使用的连接 ID的RETIRE_CONNECTION_ID 帧或 3 个 RTO 的 RETIRE_CONNECTION_ID 帧。

6. PATH_ABANDON 帧向接收对等方指示发送方不再打算在该路径上发送任何数据包，PATH_ABANDON 帧的接收者也可以发送一个 PATH_ABANDON 帧来表示它自己愿意不再在这条路径上发送任何数据包。

7. PATH_ABANDON 帧可以在任何路径上发送，而不仅仅是在要关闭的路径上发送。如果在废弃路径上发送并被认为丢失的可重传帧应该在其他路径上重传。

8. QUIC中如果只有一条路径存活并且服务端收到了PATH_ABANDON帧，那么服务端应该发送CONNECTION_CLOSE 帧并进入关闭状态，如果客户端在唯一一条存活路径上收到PATH_ABANDON 帧，它可能会尝试打开新路径（如果可用），并且仅在路径验证失败或从服务器接收到 CONNECTION_CLOSE 帧时才启动连接关闭。

9. 在路径验证过程中，端点可以通过不发送PATH_RESPONSE帧来拒绝对等方发起的新路径建立。

![image-20230213233523206](/Users/dawn/Library/Application Support/typora-user-images/image-20230213233523206.png)

#### 路径的状态管理

端点使用 PATH_STATUS 帧来通知对等方应该按照这些帧表达的偏好发送数据包。需要注意的是，端点可能不遵循对等方的通告

PATH_STATUS 帧描述了路径的两种状态：

- 将路径标记为“可用”，即允许在当前路径上发送流量。
- 将路径标记为“备用”，即建议如果另一条路径可用，则不应在该路径上发送流量。

端点使用 PATH_STATUS 帧中的路径标识符字段来标识哪个路径的状态将被更改。PATH_STATUS 帧可以通过不同的路径发送。如果端点收到的路径状态帧会使所有路径都不可用，那么端点可以忽略PATH_STATUS帧。

PATH_STATUS利用PATH_ID来标识它希望改变哪个路径的状态。另外利用path-status-seq-num来标记PATH_STATUS的有效性：

1. 这是个递增的序号，只有收到更高序号的PATH_STATUS才判定为有效
2. 不同路径上的序号不互相干涉，也就是说如果收到path-1上的高序号，不会去影响path2的状态。

![image-20230213233905630](/Users/dawn/Library/Application Support/typora-user-images/image-20230213233905630.png)

#### 路径的拥塞控制

1. 每个路径需要为每个路径都维护一个独立的拥塞状态
2. mpquic连接和普通quic连接并不使用统一的拥塞控制机制，因为这可能导致带宽分配不均，带宽会被倾向分配给多路径的connection。
3. 协议推荐在mpquic下使用LIA拥塞控制机制：RFC6356

### 参考文档

- [RFC9000中译：QUIC传输协议](https://autumnquiche.github.io/RFC9000_Chinese_Translation)
- [Multipath Extension for QUIC](https://www.ietf.org/archive/id/draft-ietf-quic-multipath-02.html)
