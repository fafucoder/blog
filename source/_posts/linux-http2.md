---
title: http1.x，http2.0, http3.0功能起底
date: 2020-09-14 19:52:55
tags:
- linux
categories:
- linux
---

### HTTP协议

[超文本传输协议](http://zh.wikipedia.org/wiki/超文本传输协议)（英文：HyperText Transfer Protocol，缩写：HTTP）是互联网上应用最为广泛的一种网络协议。设计HTTP最初的目的是为了提供一种发布和接收HTML页面的方法。通过HTTP或者HTTPS协议请求的资源由统一资源标识符（URI）来标识。

HTTP是一个应用层协议，由请求和响应构成，是一个标准的客户端服务器模型。HTTP是基于TCP/IP的无连接，无状态的协议(每一个HTTP报文不依赖于其前面的报文状态)。HTTP假定其下层协议提供可靠的传输。因此也就是其在TCP/IP协议族使用TCP作为其传输层。

 HTTP协议的主要特点可概括如下：

 简单：客户向服务器请求服务时，只需传送请求方法和路径。请求方法常用的有GET、HEAD、POST。每种方法规定了客户与服务器联系的不同类型。

 灵活：HTTP允许传输任意类型的数据对象。正在传输的类型由Content-Type加以标记。

 请求-响应模式：客户端每次向服务器发起一个请求时都建立一个连接， 服务器处理完客户的请求即断开连接。

 无状态：HTTP协议是无状态协议。无状态是指协议对于事务处理没有记忆能力。缺少状态意味着如果后续处理需要前面的信息，则它必须重传。

![HTTP协议](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1gk9uxd1u58j30uy0manky.jpg)

### HTTP工作过程

HTTP由请求和响应构成，是一个标准的客户端服务器模型（B/S）。HTTP协议永远都是客户端发起请求，服务器回送响应

1、客户机（浏览器）主动向服务器（web  server)发出连接请求(这一步骤可能需要DNS解析协议来获取服务器的IP地址)。 

2、服务器接受连接请求并建立起连接。 (1,2步即我们所熟知的TCP三次握手)

3、客户机通过此连接向服务器发出GET等http命令，(“HTTP请求报文”)。

4、服务器接到命令并根据命令向客户机传送相应的数据，(“HTTP响应报文”)。

5、客户机接收从服务器送过来的数据。

6、服务器发送完数据后，主动关闭此次连接。 （”TCP四次分手“）。

概况起来就是 客户/服务器传输过程可分为四个基本步骤：

 1) 浏览器与服务器建立连接； (TCP三次握手)

 2) 浏览器向服务器HTTP请求报文；

 3) 服务器响应浏览器请求；

 4) 断开连接。（”TCP四次分手“）

### HTTP1.X 原理

在http1.0的工作过程中，http传输数据时都要经历三次握手，四次挥手，增加了延时

在http1.1加入了keep-alive来保持tcp连接不断开，减少了部分延时

#### HTTP 1.x存在的问题

- 连接无法复用

  连接无法复用会导致每次请求都经历三次握手和慢启动。三次握手在高延迟的场景下影响较明显，慢启动则对大量小文件请求影响较大。

- 队头堵塞HOLB

  队头堵塞Head-Of-Line Blocking（HOLB）导致带宽无法被充分利用，以及后续健康请求被阻塞。[HOLB](http://stackoverflow.com/questions/25221954/spdy-head-of-line-blocking)是指一系列包（package）因为第一个包被阻塞；当页面中需要请求很多资源的时候，HOLB（队头阻塞）会导致在达到最大请求数量时，剩余的资源需要等待其他资源请求完成后才能发起请求。针对队头阻塞,人们尝试过以下办法来解决:

1. HTTP 1.0：下个请求必须在前一个请求返回后才能发出，`request-response`对按序发生。显然，如果某个请求长时间没有返回，那么接下来的请求就全部阻塞了。

2. HTTP 1.1：尝试使用 pipeling 来解决，即浏览器可以一次性发出多个请求（同个域名，同一条 TCP 链接）。但 pipeling 要求返回是按序的，那么前一个请求如果很耗时（比如处理大图片），那么后面的请求即使服务器已经处理完，仍会等待前面的请求处理完才开始按序返回。所以，pipeling 只部分解决了 HOLB。

3. 引入雪碧图、将小图内联等

   ![pipeline](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1gk9ycgutnoj318m0jy19z.jpg)

- 请求头开销大

  由于报文Header一般会携带"User Agent""Cookie""Accept""Server"等许多固定的头字段（如下图），多达几百字节甚至上千字节,但Body却经常只有几十字节. Header 内容大在一定程度上增加了传输的成本.

  ![请求头](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1gk9yfe08slj314m0cojyr.jpg)

- 安全因素

  HTTP 1.x在传输数据时，所有传输的内容都是明文，客户端和服务器端都无法验证对方的身份，这在一定程度上无法保证数据的安全性。

- 不支持服务端推送消息

  因为 HTTP/1.x 并没有推送机制。所以通常两种做法， 一是轮询，客户端定时轮询，二是websocket

### 2.0 工作原理

2015年，HTTP/2 发布。HTTP/2是现行HTTP协议（HTTP/1.x）的替代，但它不是重写，HTTP方法/状态码/语义都与HTTP/1.x一样。**HTTP/2基于SPDY，专注于性能，最大的一个目标是在用户和网站间只用一个连接（connection）**。

#### HTTP2解决的问题

针对http1.x存在的问题，http2针对的解决方案如下：

![http2解决的问题](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1gka0ebhh3jj30n80r840t.jpg)

- 二进制传输

  HTTP/2 采用二进制格式传输数据，而非 HTTP 1.x 的文本格式，二进制协议解析起来更高效。 HTTP / 1 的请求和响应报文，都是由起始行，首部和实体正文（可选）组成，各部分之间以文本换行符分隔。HTTP/2 将请求和响应数据分割为更小的帧，并且它们采用二进制编码。

  在了解 HTTP/2 之前，需要知道一些通用术语：

  1. Stream： 一个双向流，一条连接可以有多个 streams。

  2. Message： 也就是逻辑上面的 request，response。

  3. Frame:：数据传输的最小单位。每个 Frame 都属于一个特定的 stream 或者整个连接。一个 message 可能有多个 frame 组成。Frame 是 HTTP/2 里面最小的数据传输单位，一个 Frame 定义如下:

     ![Frame定义](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1gka00fnop6j30wi09g751.jpg)

     Length：也就是 Frame 的长度，默认最大长度是 16KB，如果要发送更大的 Frame，需要显式的设置 max frame size。

     Type：Frame 的类型，譬如有 DATA，HEADERS，PRIORITY 等。

     Flag 和 R：保留位。

     Stream Identifier：标识所属的 stream，如果为 0，则表示这个 frame 属于整条连接。

     Frame Payload：根据不同 Type 有不同的格式。

  Stream 有很多重要特性：

  1. 一条连接可以包含多个 streams，多个 streams 发送的数据互相不影响。

  2. Stream 可以被 client 和 server 单方面或者共享使用。

  3. Stream 可以被任意一段关闭。

  4. Stream 会确定好发送 frame 的顺序，另一端会按照接受到的顺序来处理

  5. Stream 用一个唯一 ID 来标识。如果是 client 创建的 stream，ID 就是奇数，如果是 server 创建的，ID 就是偶数。

  ![http二进制传输](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1gka0833alwj310o0rajuy.jpg)

  

  二进制传输把TCP协议的部分特性挪到了应用层，把原来的"Header+Body"的消息"打散"为数个小片的二进制"帧"(Frame),用"HEADERS"帧存放头数据、"DATA"帧存放实体数据。HTP/2数据分帧后"Header+Body"的报文结构就完全消失了，协议看到的只是一个个的"碎片"。

  HTTP/2 中，同域名下所有通信都在单个连接上完成，该连接可以承载任意数量的双向数据流。每个数据流都以消息的形式发送，而消息又由一个或多个帧组成。多个帧之间可以乱序发送，根据帧首部的流标识可以重新组装。

  ![二进制传输](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1gka0avvkx0j30w40qgjxs.jpg)

  

  Header压缩

  在 HTTP/1 中，由于使用文本的形式传输 header，在 header 携带 cookie和user agent 的情况下，可能每次都需要重复传输几百到几千的字节。HTTP/2没有使用传统的压缩算法，而是开发了专门的"HPACK”算法，在客户端和服务器两端建立“字典”，用索引号表示重复的字符串，采用哈夫曼编码来压缩整数和字符串，可以达到50%~90%的高压缩率。

  具体如下：

  1. HTTP/2 在客户端和服务器端使用“首部表”来跟踪和存储之前发送的键－值对，对于相同的数据，不再通过每次请求和响应发送；

  2. 首部表在 HTTP/2 的连接存续期内始终存在，由客户端和服务器共同渐进地更新;

  3. 每个新的首部键－值对要么被追加到当前表的末尾，要么替换表中之前的值![请求头压缩](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1gk9zw59f1mj312o0nknaq.jpg)

- 多路复用(请求和响应复用)

  同个域名只需要占用一个 TCP 连接， 在一个 TCP 连接上，我们可以向对方不断发送帧，每帧的 stream identifier 的标明这一帧属于哪个流，然后在对方接收时，根据 stream identifier 拼接每个流的所有帧组成一整块数据。

  上面协议解析中提到的stream id就是用作连接共享机制的.一个request对应一个stream并分配一个id，这样一个连接上可以有多个stream，每个stream的frame可以随机的混杂在一起，接收方可以根据stream id将frame再归属到各自不同的request里面。请求和响应之间互相不影响

  多路复用（MultiPlexing），这个功能相当于是长连接的增强，每个request可以随机的混杂在一起，接收方可以根据request的id将request再归属到各自不同的服务端请求里面。

  多路复用中，支持流的优先级（Stream dependencies），允许客户端告诉server哪些内容是更优先级的资源，可以优先传输。

  ![多路复用](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1gk9zpmtqkwj31500kcn44.jpg)

- 服务器端推送

  HTTP/2 新增的另一个强大的新功能是，服务器可以对一个客户端请求发送多个响应。 换句话说，除了对最初请求的响应外，服务器还可以向客户端推送额外资源，而无需客户端明确地请求。

  服务端可以主动推送，客户端也有权利选择是否接收。如果服务端推送的资源已经被浏览器缓存过，浏览器可以通过发送RST_STREAM帧来拒收。

  主动推送遵守同源策略, 服务器不能随便将第三方资源推送给客户端，而必须是经过双方确认才行。

  ![服务器端推送](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1gk9z83ptiij31580f2gsm.jpg)

- 提高安全性

  HTTP/2延续了HTTP/1的“明文”特点，可以像以前一样使用明文传输数据，不强制使用加密通信，不过格式还是二进制，只是不需要解密。

  由于HTTPS已经是大势所趋，而且主流的浏览器Chrome、Firefox等都公开宣布只支持加密的HTTP/2，所以“事实上”的HTTP/2是加密的。也就是说，互联网上通常所能见到的HTTP/2都是使用"https”协议名，跑在TLS上面。HTTP/2协议定义了两个字符串标识符：“h2"表示加密的HTTP/2，“h2c”表示明文的HTTP/2。

  ![http安全性](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1gk9z4p8vgqj30x80ek43h.jpg)

#### HTTP2存在的问题

1. http1.x和http2都是基于tcp, tcp作为安全的可靠的传输协议，建立连接的延时还是有点高。主要是TCP 以及 TCP+TLS建立连接的延时。

   ![tcp协议](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1gka0gjm5txj30zq0oymzy.jpg)

2. 无法解决tcp的队头堵塞问题，http2通过多路复用解决http层的对头堵塞，tcp的失败重传的问题还是无解的

   ![队头堵塞](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1gka0hybz4hj311i0ckgpb.jpg)

### HTTP3.0 原理

因为TCP 存在的时间实在太长，已经充斥在各种设备中，并且这个协议是由操作系统实现的，更新起来不大现实。

![改造tcp](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1gka0opr442j30u80km0vy.jpg)

基于这个原因，**Google 就更起炉灶搞了一个基于 UDP 协议的 QUIC 协议，并且使用在了 HTTP/3 上**，HTTP/3 之前名为 HTTP-over-QUIC，从这个名字中我们也可以发现，HTTP/3 最大的改造就是使用了 QUIC。QUIC（Quick UDP Internet Connections，快速 UDP 网络连接） 基于 UDP, 正是看中了 UDP 的速度与效率。同时 QUIC 也整合了 TCP、TLS 和 HTTP/2 的优点，并加以优化。

![QUIC](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1gka0qi7018j313g0dyted.jpg)

QUIC的次要目标包括减少连接和传输延迟，在每个方向进行带宽估计以避免拥塞。它还将拥塞控制算法移动到用户空间，而不是内核空间，此外使用前向纠错（FEC）进行扩展，以在出现错误时进一步提高性能。

#### HTTP3实现的功能

- 实现快速握手功能

  HTTP/2 的连接需要 3 RTT，如果考虑会话复用，即把第一次握手算出来的对称密钥缓存起来，那么也需要 2 RTT，更进一步的，如果 TLS 升级到 1.3，那么 HTTP/2 连接需要 2 RTT，考虑会话复用则需要 1 RTT。而 HTTP/3 首次连接只需要 1 RTT，后面的连接更是只需 0 RTT。

  ![快速握手](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1gka1dgmgifj30ys0iqdmc.jpg)

  使用QUIC协议的客户端和服务端要使用1RTT进行**密钥交换**，使用的交换算法是DH(Diffie-Hellman)**迪菲-赫尔曼算法**。

  首次连接时**客户端和服务端的密钥协商和数据传输过程**，其中涉及了DH算法的基本过程：

  1. 客户端对于首次连接的服务端先发送client hello请求。

  2. 服务端生成一个素数p和一个整数g，同时生成一个随机数为私钥a，然后计算出公钥A = mod p，服务端将A，p，g三个元素打包称为config，后续发送给客户端。

  3. 客户端随机生成一个自己的私钥b，再从config中读取g和p，计算客户端公钥B = mod p。然后客户端根据 A、p、b 算出初始密钥 K。B 和 K 算好后，客户端会用 K 加密 HTTP 数据，连同 B 一起发送给服务端；

  4. 客户端使用自己的私钥b和服务端发来的config中读取的服务端公钥，生成后续数据加密用的密钥K = mod p。

  5. 客户端使用密钥K加密业务数据，并追加自己的公钥，都传递给服务端。

  6. 服务端根据自己的私钥a和客户端公钥B生成客户端加密用的密钥K = mod p。

  7. 为了保证数据安全，上述生成的密钥K只会生成使用1次，后续服务端会按照相同的规则生成一套全新的公钥和私钥，并使用这组公私钥生成新的密钥M。

  8. 服务端将新公钥和新密钥M加密的数据发给客户端，客户端根据新的服务端公钥和自己原来的私钥计算出本次的密钥M，进行解密。

  9. 之后的客户端和服务端数据交互都使用密钥M来完成，密钥K只使用1次。

     ![交互过程](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1gka1n5fcntj30yk0isn16.jpg)

     ![交互过程](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1gka1o5mhntj30vn0u0wlp.jpg)

  

  客户端和服务端首次连接时服务端传递了**config包**，里面包含了服务端公钥和两个随机数，**客户端会将config存储下来**，后续再连接时可以直接使用，从而跳过这个1RTT，实现0RTT的业务数据交互。

  客户端保存config是有时间期限的，在config失效之后仍然需要进行首次连接时的密钥交换。

- 集成TLS加密功能

  1. 向前安全

     **前向安全指的是密钥泄漏也不会让之前加密的数据被泄漏，影响的只有当前，对之前的数据无影响**。

     QUIC使用config的有效期，后续又生成了新密钥，使用其进行加解密，当时完成交互时则销毁，从而实现了前向安全。

     ![向前安全](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1gka1vsoi56j310a0ogjxn.jpg)

  2. 向前纠错

     前向纠错也叫前向纠错码Forward Error Correction 简称FEC 是增加数据通讯可信度的方法，在单向通讯信道中，一旦错误被发现，其接收器将无权再请求传输。

     FEC 是利用数据进行传输冗余信息的方法，当传输中出现错误，将允许接收器再建数据。

     QUIC每发送一组数据就对这组数据进行**异或运算**，并将结果作为一个FEC包发送出去，接收方收到这一组数据后根据数据包和FEC包即可进行校验和纠错。一段数据被切分为 10 个包后，依次对每个包进行异或运算，运算结果会作为 FEC 包与数据包一起被传输，如果不幸在传输过程中有一个数据包丢失，那么就可以根据剩余 9 个包以及 FEC 包推算出丢失的那个包的数据，这样就大大增加了协议的容错性。

     ![向前纠错](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1gka1ybqivbj30yw0j6ad5.jpg)

- 多路复用，解决tcp中存在的队头堵塞问题

  HTTP2.0协议的**多路复用机制**解决了HTTP层的队头阻塞问题，但是在**TCP层仍然存在队头阻塞问题**。

  TCP协议在收到数据包之后，这部分数据可能是乱序到达的，但是TCP必须将所有数据收集排序整合后给上层使用，**如果其中某个包丢失了，就必须等待重传，从而出现某个丢包数据阻塞整个连接的数据使用**。

  QUIC协议是基于UDP协议实现的，在一条链接上可以有**多个流**，流与流之间是**互不影响**的，当一个流出现丢包影响范围非常小，从而解决队头阻塞问题。

  ![队头堵塞](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/0081Kckwly1gka233ve3wj30zi0jomzv.jpg)

- 实现类似TCP的流量控制(...这部分内容多，看参考)

### 参考文档

- [五分钟读懂http3](https://mp.weixin.qq.com/s/T8848OB4EWCIdk7gFfRLbQ)
- [HTTP/3 原理实战](https://mp.weixin.qq.com/s?__biz=MzI1OTY2MzMxOQ==&mid=2247488320&idx=1&sn=0beb1b90a0318f526e290b22ee16f220)
- [图解为什么http3使用udp协议](https://mp.weixin.qq.com/s/SrGpxKqP4JpEuHoebsLWEA)
- [科普：QUIC协议原理分析](https://zhuanlan.zhihu.com/p/32553477)
- [解密http/2与http/3 的特性](https://segmentfault.com/a/1190000020714686) //http2,http3
- [grpc之http2协议](https://www.jianshu.com/p/9e575496bf8a) //http2
- [http协议解析](https://blog.csdn.net/daniel_ustc/article/details/17955005) //http
