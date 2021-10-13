---
title: nodejs中require跟import的区别
date: 2021-10-12 21:25:22
tags:
- nodejs
categories:
- nodejs
---

### 概述

>  require/exports 出生在野生规范当中，什么叫做野生规范？即这些规范是 JavaScript 社区中的开发者自己草拟的规则，得到了大家的承认或者广泛的应用。比如 CommonJS、AMD、CMD 等等。import/export 则是名门正派。TC39 制定的新的 ECMAScript 版本，即 ES6（ES2015）中包含进来。

关于 `import` 和 `require` 的不同，其实可以理解成 CommonJs 和 ES Module 的区别。这两者都是前端模块化的规范。

#### CommonJs

Nodejs 是 CommonJS 规范的主要实践者，在 CommonJs 里每个文件就是一个模块，有自己的作用域。在一个文件里面定义的变量、函数、类，都是私有的，对其他文件不可见。

```javascript
class MyClass {
  constructor(name) {
    this.name = name;
  }
}

module.exports.MyClass = Myclass

const MyClass = require('./myclass.js')
const obj = new MyClass('hello')
```

#### ES Module

ES6 模块的设计思想是尽量的静态化，使得编译时就能确定模块的依赖关系，以及输入和输出的变量。所以ES6 模块不是对象，而是通过 `export` 命令显式指定输出的代码，再通过 `import` 命令输入。这种加载称为“编译时加载”或者静态加载，即 ES6 可以在编译时就完成模块加载，效率要比 CommonJS 模块的加载方式高。

```javascript
export default class MyClass {
	constructor(name) {
    this.name = name;
  }
}

import MyClass from './myclass'
const obj = new MyClass('hello')
```

### 区别

| 命令    | 规范           | 调用       | 本质     | 特点                                                         |
| ------- | -------------- | ---------- | -------- | ------------------------------------------------------------ |
| require | CommonJS规范   | 运行时调用 | 赋值过程 | 非语言层面的标准。 社区方案，提供了服务器/浏览器的模块加载方案。只能在运行时确定模块的依赖关系及输入/输出的变量，无法进行静态优化。 |
| import  | es6+的语法标准 | 编译时调用 | 解构过程 | 语言规格层面支持模块功能。支持编译时静态分析，便于JS引入宏和类型检验。动态绑定 |

### 关于调用

1. require的引用可以在代码的任何地方。
2. import语法规范上是放在文件开头。

### 参考文档

- https://juejin.cn/post/6844903765489745927 // 值得一看
- https://www.zhihu.com/question/56820346     // 值得一看
- https://segmentfault.com/a/1190000023082896  // 值得一看