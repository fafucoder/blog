---
title: web应用的未来是webAssembly?
date: 2021-10-13 21:30:10
tags:
- golang
categories:
- golang
---

## 什么是WebAssembly

在说明什么是Webassembly之前，我们有必要了解一下asm.js。2012年，Mozilla 的工程师 Alon Zakai 在研究 LLVM 编译器时突发奇想：许多 3D 游戏都是用 C / C++ 语言写的，如果能将 C / C++ 语言编译成 JavaScript 代码，它们不就能在浏览器里运行了吗？众所周知，JavaScript 的基本语法与 C 语言高度相似。于是，他开始研究怎么才能实现这个目标，为此专门做了一个编译器项目 Emscripten。这个编译器可以将 C / C++ 代码编译成 JS 代码，但不是普通的 JS，而是一种叫做 asm.js 的 JavaScript 变体，性能差不多是原生代码的50%。

之后Google开发了Portable Native Client，也是一种能让浏览器运行C/C++代码的技术。 后来可能是因为彼此之间有共同的更高追求，Google, Microsoft, Mozilla, Apple等几家大公司一起合作开发了一个面向Web的通用二进制和文本格式的项目，那就是WebAssembly。asm.js 与 WebAssembly 功能基本一致，就是转出来的代码不一样：asm.js 是文本，WebAssembly 是二进制字节码，因此运行速度更快、体积更小。

WebAssembly(又称 wasm) 是一种新的字节码格式，主流浏览器都已经支持 WebAssembly。 和 JS 需要解释执行不同的是，WebAssembly 字节码和底层机器码很相似可快速装载运行，因此性能相对于 JS 解释执行大大提升。 也就是说 WebAssembly 并不是一门编程语言，而是一份字节码标准，需要用高级编程语言编译出字节码放到 WebAssembly 虚拟机中才能运行， 浏览器厂商需要做的就是根据 WebAssembly 规范实现虚拟机。

这里引用MDN上官方对其的解释：WebAssembly是一种新的编码方式，可以在现代的网络浏览器中运行 － 它是一种低级的类汇编语言，具有紧凑的二进制格式，可以接近原生的性能运行，并为诸如C / C ++ / Rust等语言提供一个编译目标，以便它们可以在Web上运行。它也被设计为可以与JavaScript共存，允许两者一起工作(总结一句话：Webassembly就像是JVM虚拟机，可以执行二进制码)。

> 有人这么评价Webassembly： Webassembly是Rust押宝的唯一使用场景(Rust语言的使用场景太少了，不像golang在云原生跟中间键领域有很多应用)。

## 为什么需要WebAssembly

自从 JavaScript 诞生起到现在已经变成最流行的编程语言，这背后正是 Web 的发展所推动的。Web 应用变得更多更复杂，但这也渐渐暴露出了 JavaScript 的问题：

1. 语法太灵活导致开发大型 Web 项目困难；
2. 性能不能满足一些场景的需要。

针对以上两点缺陷，近年来出现了一些 JS 的代替语言，例如：

1. 微软的[ TypeScript ](http://www.typescriptlang.org/)通过为 JS 加入静态类型检查来改进 JS 松散的语法，提升代码健壮性；
2. 谷歌的[ Dart ](https://www.dartlang.org/)则是为浏览器引入新的虚拟机去直接运行 Dart 程序以提升性能；
3. 火狐的[ asm.js ](http://asmjs.org/)则是取 JS 的子集，JS 引擎针对 asm.js 做性能优化。

以上尝试各有优缺点，其中：

1. TypeScript 只是解决了 JS 语法松散的问题，最后还是需要编译成 JS 去运行，对性能没有提升；
2. Dart 只能在 Chrome 预览版中运行，无主流浏览器支持，用 Dart 开发的人不多；
3. asm.js 语法太简单、有很大限制，开发效率低。

三大浏览器巨头分别提出了自己的解决方案，互不兼容，这违背了 Web 的宗旨； 是技术的规范统一让 Web 走到了今天，因此形成一套新的规范去解决 JS 所面临的问题迫在眉睫。于是 WebAssembly 诞生了，WebAssembly 是一种新的字节码格式，主流浏览器都已经支持 WebAssembly。 和 JS 需要解释执行不同的是，WebAssembly 字节码和底层机器码很相似可快速装载运行，因此性能相对于 JS 解释执行大大提升。 也就是说 WebAssembly 并不是一门编程语言，而是一份字节码标准，需要用高级编程语言编译出字节码放到 WebAssembly 虚拟机中才能运行， 浏览器厂商需要做的就是根据 WebAssembly 规范实现虚拟机。

> 总结一句话就是：主流厂商极力主导 + webassembly速度快

## Webassembly原理

要搞懂 WebAssembly 的原理，需要先搞懂计算机的运行原理。 电子计算机都是由电子元件组成，为了方便处理电子元件只存在开闭两种状态，对应着 0 和 1，也就是说计算机只认识 0 和 1，数据和逻辑都需要由 0 和 1 表示，也就是可以直接装载到计算机中运行的机器码。 机器码可读性极差，因此人们通过高级语言 C、C++、Rust、Go 等编写再编译成机器码。

由于不同的计算机 CPU 架构不同，机器码标准也有所差别，常见的 CPU 架构包括 x86、AMD64、ARM， 因此在由高级编程语言编译成可自行代码时需要指定目标架构。WebAssembly 字节码是一种抹平了不同 CPU 架构的机器码，WebAssembly 字节码不能直接在任何一种 CPU 架构上运行， 但由于非常接近机器码，可以非常快的被翻译为对应架构的机器码，因此 WebAssembly 运行速度和机器码接近，这听上去非常像 Java 字节码。

在javascript中，JavaScript引擎是执行 JavaScript 代码的程序或解释器。JavaScript引擎可以实现为标准解释器，或者以某种形式将JavaScript编译为字节码的即时编译器。不用的浏览器使用不同的引擎如下所示：

- [V8](https://link.segmentfault.com/?enc=sUDGcZBE4XE+vSy5zjw84Q==.NqvwmK7YdB1NXJpJlh2DNQLg1aBzu82zEWbf/mLpt9JyjRr3+KoxkLT+FwB8JUSoRUdZr2njfFAdW/ex3PcpnA==) — 开源，由 Google 开发，用 C ++ 编写

- [Rhino](https://link.segmentfault.com/?enc=Lcfwj3jXwKG3LmIWG3zMdA==.96YdUZxH/6IBmMcriO6tSIZNC1VIRPZ/eQZKc278Lf6Hoz1pxLZVOwjHu0/qcOMVqZ9l7XET4ID0V4QgJc/f7w==) — 由 Mozilla 基金会管理，开源，完全用 Java 开发

- [SpiderMonkey](https://link.segmentfault.com/?enc=fEBfBX1Wd1oq35Opx9XqBg==.+OCUcVZ4OMlh0scYGO+X5yDy31T3RhLgz5LU8pR/QG3GrOaSb2UWZAqEPjXXQxNg) — 是第一个支持 Netscape Navigator 的 JavaScript 引擎，目前正供 Firefox 使用

- [JavaScriptCore](https://link.segmentfault.com/?enc=5dW2eusqpsCf0enmAMJYHg==.ecsPVoKRqxuA7xY07xwHk2pUZj9fu69Y5Xk9MFLPPcYUhCtmLH8aLW7GfggLyLZB5Fn3g7TYJR2k0t3941/3mA==) — 开源，以Nitro形式销售，由苹果为Safari开发

- [KJS](https://link.segmentfault.com/?enc=wE8dCPOCitI59es0P62axw==.NQQ7Mbn0KzdeG2aWMep/ZoMepjYQU0i2Y70FT0ArpGB6zy+1ygTmPYQDKCYBwp3bzBw4MzXWC2RvNNsv1WCpUg==) — KDE 的引擎，最初由 Harri Porten 为 KDE 项目中的 Konqueror 网页浏览器开发

- [Chakra](https://link.segmentfault.com/?enc=4p1lBDVXg9F3LoYDCAYqZw==.69EhewNej7/0qL6i/vfOmKrHRr8J75bQnL6m3/Pq/NJ9B/5ltxTPYbJTUIhqMwAMHFWYhsMI6zbTzo/cs4QMyA==) (JScript9) — Internet Explorer

- [Chakra](https://link.segmentfault.com/?enc=HbF4R5UwNZvgHmR/1wtgDQ==.eZWlURXx88Zj4cbJlV52kYcs6ilppQjxqB9f0e/bKPr6736fvUxWXbL77DDy7W8ONHasBxCyxiS/G07KFi3mIQ==) (JavaScript) — Microsoft Edge

- [Nashorn](https://link.segmentfault.com/?enc=JJkYtp97F7aGkuRpJu4j+A==.m2XjtFgQZ+p7pfw2mm6hzwz8hHSy2krktwpxq8U1xVjUx1swsJd9bpTYlTiR4YQF5iEyx+frqx1Ir7CGq4Yi8A==), 作为 OpenJDK 的一部分，由 Oracle Java 语言和工具组编写

- [JerryScript](https://link.segmentfault.com/?enc=s1goWbasfcZ7FRTX379/UA==.utRib33Njx3A6/klirW/KCvy0haJmdcHrB0fRetgT6Kg+zz1lM2n0QTZNDYWTvPE) — 物联网的轻量级引擎

因此我们看下javascript在V8引擎下的工作机制: javacript运行中，引擎首先解析javascript生成抽象语法树(AST), 然后生成机器码。

![img](https://i7drsi3tvf.feishu.cn/space/api/box/stream/download/asynccode/?code=M2NkMjI2MTBlMDY5ZjIzNzQxMzk2NWMxYzI3ZjQwNDFfSUhEV1hsbGlFRFl5bU5xTUs1NEJKV3pjU2ZBZTFRWWNfVG9rZW46Ym94Y25iNERHS1doN25BNkxpdGE2dGdVaDdSXzE2MzQxMzE5MDM6MTYzNDEzNTUwM19WNA)

接着通过V8 的优化编译器 (TurboFan)的优化，把优化后的机器码推向后端(所谓的后端就是即时编译器JIT)

![img](https://i7drsi3tvf.feishu.cn/space/api/box/stream/download/asynccode/?code=ODlmMmQ3ZDBmMDdmYTgyYmE5MGM0YWI2MjdmMzdjNDBfbXNsRXN5TThZMU9mdGJPYU42eVFBd2xrQXY0RjRrTnNfVG9rZW46Ym94Y256MFZjSGFHZHUwc3VoU2M5WG4xYkxpXzE2MzQxMzE5MDM6MTYzNDEzNTUwM19WNA)

但是webassembly并不需要以上的全部步骤－如下所示是它被插入到执行过程示意图:

![img](https://i7drsi3tvf.feishu.cn/space/api/box/stream/download/asynccode/?code=MzlhZTAzZjExYzVhYjFmZTY2MGJhZDdiNTFmNWM0MWFfUTVrRVZuMWxhMVlpMTcwN3lDVmJaYVV4WHZaVm5oNVpfVG9rZW46Ym94Y25aUEFUWlAxUm44SjJjbzZjQ3NUREdnXzE2MzQxMzE5MDM6MTYzNDEzNTUwM19WNA)

可以看到webassembly并不需要编译阶段，而是直接把编译后的字节码发送给后端，因此速度更快。

目前能编译成 WebAssembly 字节码的高级语言有：

- [AssemblyScript](https://github.com/AssemblyScript/assemblyscript):语法和 TypeScript 一致，对前端来说学习成本低，为前端编写 WebAssembly 最佳选择；
- C\C++:官方推荐的方式，详细使用见[文档](http://webassembly.org.cn/getting-started/developers-guide/);

- [Rust](https://www.rust-lang.org/):语法复杂、学习成本高，详细使用见[文档](https://github.com/rust-lang-nursery/rust-wasm);

- [Kotlin](http://kotlinlang.org/):语法和 Java、JS 相似，语言学习成本低，详细使用见[文档](https://kotlinlang.org/docs/reference/native-overview.html);

- [Golang](https://golang.org/):语法简单学习成本低。详细使用见[文档](https://blog.gopheracademy.com/advent-2017/go-wasm/)。

- 其他语言: 详细见[文档](https://github.com/appcypher/awesome-wasm-langs)。

## Webassembly使用场景：

总体而言，WASM不会替代JavaScript，但是却可以辅助解决很多JavaScript无法解决的问题。下面是Webassembly的一些场景：

##### 浏览器场景：

- 更好的让一些语言和工具可以编译到 Web 平台运行。

- 图片/视频编辑。

- 游戏：
  - 需要快速打开的小游戏

- AAA 级，资源量很大的游戏。

- 游戏门户（代理/原创游戏平台）

- P2P 应用（游戏，实时合作编辑）

- 音乐播放器（流媒体，缓存）

- 图像识别

- 视频直播

- VR 和虚拟现实

- CAD 软件

- 科学可视化和仿真

- 互动教育软件和新闻文章。

- 模拟/仿真平台(ARC, DOSBox, QEMU, MAME, …)。

- 语言编译器/虚拟机。

##### 非浏览器场景：

- 游戏分发服务（便携、安全）。

- 服务端执行不可信任的代码。

- 服务端应用。

- 移动混合原生应用。

- 多节点对称计算

## 下一步

基于golang实现webassembly的demo