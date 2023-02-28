---
title: golang代码测试方法论
date: 2023-02-25 18:45:20
tags:
- golang
categories:
- golang
---

### 概述

在项目开发中，正对项目的测试分为多个分类，比如单元测试、集成测试、端到端测试等，按照Mike Cohn提出的“测试金字塔”概念，测试分为4个层次：

![test](https://i.loli.net/2019/05/21/5ce34ea15584577479.png)

最下面的是单元测试，单元测试针对的是代码进行测试，针对的是局部代码功能；再而之上是集成测试，它针对的是服务的接口进行测试；接着网上是端到端的测试，也就是我们所说的链路测试，它针对的是服务的功能进行测试，负责从一个链路的入口输入测试用例，验证输出的系统的结果；再上一层是我们最常用的UI测试，就是测试人员在UI界面上根据功能进行点击测试。

测试金字塔建议我们：

（1）**尽可能地多做单元测试 和 集成测试**，因为他们的执行速度相较于上层的几个测试类型来说快很多且相对稳定，可以一天多次执行。

（2）**尽可能地少做 组件测试、端到端测试 和 探索性测试**，因为他们的执行速度相较单元测试 和 集成测试 会慢很多，且不够稳定，无法做到一天多次执行，每次执行都要等很久才能获得反馈结果。

> 画外音：金字塔里，越往下速度越快且越稳定，那么就可以频繁执行，反正执行一次也花不了多久时间，开发人员还可以知道我的代码有没有影响到其他模块。越往上则速度越慢且越不稳定，跑一次要N久，开发人员往往会觉得还是先继续开发吧，到时候出了bug再说，我可不想加班等测试结果。

### 单元测试

#### 什么是单元你测试

单元是应用的最小可测试部件。在过程化编程中，一个单元就是单个程序、函数、过程等；对于面向对象编程，最小单元就是方法，包括基类、超类、抽象类等中的方法。单元测试就是软件开发中对最小单位进行正确性检验的测试工作。

不同地方对单元测试有的定义可能会有所不同，但有一些基本共识：

- 单元测试是比较**底层**的，关注代码的**局部**而不是整体。
- 单元测试是开发人员在写代码时候写的。
- 单元测试需要比其他测试运行得**快**。

#### 单元测试的意义

- **提高代码质量**: 代码测试都是为了帮助开发人员发现问题从而解决问题，提高代码质量。
- **尽早发现问题**: 问题越早发现，解决的难度和成本就越低。
- **保证重构正确性**: 随着功能的增加，重构（修改老代码）几乎是无法避免的。很多时候我们不敢重构的原因，就是担心其它模块因为依赖它而不工作。有了单元测试，只要在改完代码后运行一下单测就知道改动对整个系统的影响了，从而可以让我们放心的重构代码。
- **简化调试过程**: 单元测试让我们可以轻松地知道是哪一部分代码出了问题。
- **简化集成过程**: 由于各个单元已经被测试，在集成过程中进行的后续测试会更加容易。
- **优化代码设计**: 编写测试用例会迫使开发人员仔细思考代码的设计和必须完成的工作，有利于开发人员加深对代码功能的理解，从而形成更合理的设计和结构。
- **单元测试是最好的文档**: 单元测试覆盖了接口的所有使用方法，是最好的示例代码。而真正的文档包括注释很有可能和代码不同步，并且看不懂。

#### 单元测试基本原则

1. 单元测试文件名以xxx_test.go命名，测试文件跟被测试文件处于同个包中;
2. 测试用例以Test开头, 测试函数为TestXxx,  整体风格保持一致;
3. 测试用例应该完备，应考虑到各个边界情况，推荐使用table-driven的方式实现;
4. 单元测试的结果是确定且一致的，跟环境无关的;
5. 测试用例是独立的，不同的测试用例之间不能有互相依赖;
6. 单元测试程序不应该有用户输入，测试结果应该能直接被电脑获取，不应该由人来判断;
7. 函数中通过调用testing.T的Error, Errorf, FailNow, Fatal, FatalIf等方法记录测试的信息;
8. 可以通过benchmark做并发测试，验证代码性能;

#### 常见的测试框架

golang自带的testing包已经可以完美支持单元测试，我们在写一个文件，函数的时候，可以直接在需要单元测试的文件旁边增加一个_test.go的文件。而后直接使用 `go test` 直接跑测试用例就可以了。但是因为单元测试是与环境无关的，因此在编写测试用例时总是需要把一些第三发依赖给mock掉，因此需要借助其他的测试组件来完成，下面是一些常见的mock框架：

1. 基于接口的Mock框架-goMock  
2. 全局变量、函数、过程打桩框架-goStub
3. 动态打桩框架(非并发安全)-goMonkey
4. 数据库和缓存等中间件mock框架(sqlMock, redisMock)
5. http请求mock框架-httpmock

##### goMonkey

[go monkey](https://github.com/agiledragon/gomonkey) 可以动态的执行打桩，目标是让用户在单元测试中低成本的完成打桩，从而将精力聚焦于业务功能的开发。gomonkey支持多种打桩方式，可以无入侵的实现打桩，但是需要注意的是：

1. 运行 monkey需要关闭 Go 语言的内联优化才能生效，也就是编译时需要添加 `-gcflags=all=-l` 参数才行

   > 注意：
   >
   > 1. 如果是 M1 系统，编译需要指定 goarch 为 amd64
   >    @MAC-M1#  GOARCH=amd64 go build -gcflags=all=-l  -o xxx  main.go
   >    @MAC-INTEL# go build -gcflags=all=-l  -o xxx  main.go
   >    解决的问题：monkey mac m1 execute syscall.Mprotect error：panic: permission denied [recovered]
   >    ref: https://github.com/agiledragon/gomonkey/issues/57
   >    ref: https://blog.csdn.net/qw790707988/article/details/119710144
   > 2. monkey不应该用于生产系统, 否则可能引发未知事故
   > 3. monkey 需要在运行的时候修改内存代码段，因而无法在一些对安全性要求比较高的系统上工作

### 集成测试与E2E测试

go开发者一般使用Ginkgo + Gomega 实现集成测试与E2E测试。Ginkgo /ˈɡɪŋkoʊ / 是Go语言的一个行为驱动开发（BDD， Behavior-Driven Development）风格的测试框架，通常和库Gomega一起使用。Ginkgo在一系列的“Specs”中描述期望的程序行为。

#### Ginkgo简介

inkgo是Go语言的一个行为驱动开发（BDD， Behavior-Driven Development）风格的测试框架，通常和库Gomega一起使用。Ginkgo在一系列的“Specs”中描述期望的程序行为。

##### 安装使用

```base
go get -u github.com/onsi/ginkgo/ginkgo

ginkgo bootstrap
```

##### 测试模块

- It: 是测试例的基本单位，即It包含的代码就算一个测试用例
- Context和Describe: 是将一个或多个测试例归类
- BeforeEach: 是每个测试例执行前执行该段代码
- AfterEach: 是每个测试例执行后执行该段代码
- JustBeforeEach: 是在BeforeEach执行之后，测试例执行之前执行
- BeforeSuite: 是在该测试集执行前执行，即该文件夹内的测试例执行之前
- AfterSuite: 是在该测试集执行后执行，即该文件夹内的测试例执行完后
- By: 是打印信息，内容只能是字符串，只会在测试例失败后打印，一般用于调试和定位问题
- Fail: 是标志该测试例运行结果为失败，并打印里面的信息
- Specify: 和It功能完全一样，It属于其简写

##### 测试方法列表

```bash
# 断言
Expect(ACTUAL).Should(Equal(EXPECTED))
Expect(ACTUAL).To(Equal(EXPECTED))
Expect(ACTUAL).ShouldNot(Equal(EXPECTED))
Expect(ACTUAL).NotTo(Equal(EXPECTED))
Expect(ACTUAL).ToNot(Equal(EXPECTED))

// 断言没有发生错误
Expect(err).ShouldNot(HaveOccurred())
Expect(DoSomethingSimple()).Should(Succeed())

// 断言注解
Expect(ACTUAL).To(Equal(EXPECTED), "My annotation %d", foo)
Expect(ACTUAL).To(Equal(EXPECTED), func() string { return "My annotation" })

# 相等
// 如果ACTUAL和EXPECTED都为nil，断言会失败
Expect(ACTUAL).Should(Equal(EXPECTED))
 
Expect(ACTUAL).Should(BeEquivalentTo(EXPECTED))
 
// 使用 == 进行比较
BeIdenticalTo(expected interface{})

# 空值/零值
// 断言ACTUAL为Nil
Expect(ACTUAL).Should(BeNil())
 
// 断言ACTUAL为它的类型的零值，或者是Nil
Expect(ACTUAL).Should(BeZero())

# 布尔值
Expect(ACTUAL).Should(BeTrue())
Expect(ACTUAL).Should(BeFalse())

# 错误
Expect(ACTUAL).Should(HaveOccurred())
 
// 没有错误
err := SomethingThatMightFail()
Expect(err).ShouldNot(HaveOccurred())

// 如果ACTUAL为Nil则断言成功
Expect(ACTUAL).Should(Succeed())

# 字符串
// 子串判断
Expect(ACTUAL).Should(ContainSubstring(STRING, ARGS...))
 
// 前缀判断
Expect(ACTUAL).Should(HavePrefix(STRING, ARGS...))
 
// 后缀判断
Expect(ACTUAL).Should(HaveSuffix(STRING, ARGS...))
 
// 正则式匹配
Expect(ACTUAL).Should(MatchRegexp(STRING, ARGS...))

# JSON/XML/YML
Expect(ACTUAL).Should(MatchJSON(EXPECTED))
Expect(ACTUAL).Should(MatchXML(EXPECTED))
Expect(ACTUAL).Should(MatchYAML(EXPECTED))

# 集合（string, array, map, chan, slice）
// 断言为空
Expect(ACTUAL).Should(BeEmpty())
 
// 断言长度
Expect(ACTUAL).Should(HaveLen(INT))
 
// 断言容量
Expect(ACTUAL).Should(HaveCap(INT))
 
// 断言包含元素
Expect(ACTUAL).Should(ContainElement(ELEMENT))
 
// 断言等于 其中之一
Expect(ACTUAL).Should(BeElementOf(ELEMENT1, ELEMENT2, ELEMENT3, ...))
 
// 断言元素相同，不考虑顺序
Expect(ACTUAL).Should(ConsistOf(ELEMENT1, ELEMENT2, ELEMENT3, ...))
Expect(ACTUAL).Should(ConsistOf([]SOME_TYPE{ELEMENT1, ELEMENT2, ELEMENT3, ...}))
 
// 断言存在指定的键，仅用于map
Expect(ACTUAL).Should(HaveKey(KEY))

// 断言存在指定的键值对，仅用于map
Expect(ACTUAL).Should(HaveKeyWithValue(KEY, VALUE))

# 数字/时间
// 断言数字意义（类型不感知）上的相等
Expect(ACTUAL).Should(BeNumerically("==", EXPECTED))
 
// 断言相似，无差不超过THRESHOLD（默认1e-8）
Expect(ACTUAL).Should(BeNumerically("~", EXPECTED, <THRESHOLD>))
 
 
Expect(ACTUAL).Should(BeNumerically(">", EXPECTED))
Expect(ACTUAL).Should(BeNumerically(">=", EXPECTED))
Expect(ACTUAL).Should(BeNumerically("<", EXPECTED))
Expect(ACTUAL).Should(BeNumerically("<=", EXPECTED))
 
Expect(number).Should(BeBetween(0, 10))
```

### 参考文档

- [golang项目的测试实践](https://www.cnblogs.com/yjf512/p/10905352.html)
- [What Is 测试金字塔？](https://blog.csdn.net/sd7o95o/article/details/107804441)
- [gomonkey调研文档和学习](https://blog.csdn.net/u013276277/article/details/104993370)
- [ginkgo学习笔记](https://blog.gmem.cc/ginkgo-study-note)
