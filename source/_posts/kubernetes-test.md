---
title: Ginkgo-golang BDD代码测试框架
date: 2020-02-05 11:53:26
tags:
- kubernetes
categories:
- kubernetes
---

### Ginkgo简介
inkgo是Go语言的一个行为驱动开发（BDD， Behavior-Driven Development）风格的测试框架，通常和库Gomega一起使用。Ginkgo在一系列的“Specs”中描述期望的程序行为。

### 安装使用
```base

go get -u github.com/onsi/ginkgo/ginkgo

ginkgo bootstrap

```

### 测试模块

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

### 测试方法

#### 断言
```
Expect(ACTUAL).Should(Equal(EXPECTED))
Expect(ACTUAL).To(Equal(EXPECTED))
Expect(ACTUAL).ShouldNot(Equal(EXPECTED))
Expect(ACTUAL).NotTo(Equal(EXPECTED))
Expect(ACTUAL).ToNot(Equal(EXPECTED))

// 断言没有发生错误
Expect(err).ShouldNot(HaveOccurred())
Expect(DoSomethingSimple()).Should(Succeed())

//断言注解
Expect(ACTUAL).To(Equal(EXPECTED), "My annotation %d", foo)
Expect(ACTUAL).To(Equal(EXPECTED), func() string { return "My annotation" })
```

#### 相等
```
// 如果ACTUAL和EXPECTED都为nil，断言会失败
Expect(ACTUAL).Should(Equal(EXPECTED))
 
Expect(ACTUAL).Should(BeEquivalentTo(EXPECTED))
 
// 使用 == 进行比较
BeIdenticalTo(expected interface{})
```

#### 空值/零值
```
// 断言ACTUAL为Nil
Expect(ACTUAL).Should(BeNil())
 
// 断言ACTUAL为它的类型的零值，或者是Nil
Expect(ACTUAL).Should(BeZero())
```

#### 布尔值
```
Expect(ACTUAL).Should(BeTrue())
Expect(ACTUAL).Should(BeFalse())
```
#### 错误
```
Expect(ACTUAL).Should(HaveOccurred())
 
err := SomethingThatMightFail()
// 没有错误
Expect(err).ShouldNot(HaveOccurred())
 
 
// 如果ACTUAL为Nil则断言成功
Expect(ACTUAL).Should(Succeed())
```

### 字符串
```
// 子串判断
Expect(ACTUAL).Should(ContainSubstring(STRING, ARGS...))
 
// 前缀判断
Expect(ACTUAL).Should(HavePrefix(STRING, ARGS...))
 
// 后缀判断
Expect(ACTUAL).Should(HaveSuffix(STRING, ARGS...))
 
// 正则式匹配
Expect(ACTUAL).Should(MatchRegexp(STRING, ARGS...))
```

#### JSON/XML/YML
```
Expect(ACTUAL).Should(MatchJSON(EXPECTED))
Expect(ACTUAL).Should(MatchXML(EXPECTED))
Expect(ACTUAL).Should(MatchYAML(EXPECTED))
```

#### 集合（string, array, map, chan, slice）
```
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
```

#### 数字/时间
```
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
- https://blog.gmem.cc/ginkgo-study-note
- https://onsi.github.io/ginkgo/#running-tests
- https://blog.csdn.net/goodboynihaohao/article/details/79392500