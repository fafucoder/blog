---
title: go mod整理文档
date: 2020-03-09 12:25:37
tags:
- golang
categories:
- golang
---

### GOPATH

#### 什么是GOPATH
终端输入go env
```
➜  ~ go env
GOPATH="/Users/dawn/go"
...
```

进入到GOPATH目录下结构如下
```
➜  ~ tree -d -L 2 $GOPATH
/Users/dawn/go
├── bin
├── pkg
│   ├── mod
│   └── sumdb
└── src
    ├── github.com
    ├── go-types
    ├── gomod-test
    └── k8s.io
```

GOPATH目录下一共包含了三个子目录，分别是：
- bin：存储所编译生成的二进制文件。
- pkg：存储预编译的目标文件，以加快程序的后续编译速度。
- src：存储所有.go文件或源代码。在编写 Go 应用程序，程序包和库时，一般会以$GOPATH/src/github.com/foo/bar的路径进行存放。

因此在使用 GOPATH 模式下，我们需要将应用代码存放在固定的$GOPATH/src目录下，并且如果执行go get来拉取外部依赖会自动下载并安装到$GOPATH目录下。

#### 为何弃用GOPATH
GOPATH 模式下没有版本控制的概念，具有致命的缺陷，至少会造成以下问题
- 在执行go get的时候，你无法传达任何的版本信息的期望，也就是说你也无法知道自己当前更新的是哪一个版本，也无法通过指定来拉取自己所期望的具体版本
- 在运行Go应用程序的时候，你无法保证其它人与你所期望依赖的第三方库是相同的版本，也就是说在项目依赖库的管理上，你无法保证所有人的依赖版本都一致
- 你没办法处理 v1、v2、v3 等等不同版本的引用问题，因为 GOPATH 模式下的导入路径都是一样的，都是github.com/foo/bar

因此Go语言官方从 Go1.11 起开始推进 Go modules（前身vgo），Go1.13 起不再推荐使用 GOPATH 的使用模式

### Go Modules

#### GO MOD环境配置
通过go env可以查看配置
```
➜  ~ go env                
GO111MODULE="off"
GOBIN="/Users/dawn/go/bin"
GOPATH="/Users/dawn/go"
GOPRIVATE=""
GOPROXY="https://goproxy.cn,direct"
GOROOT="/usr/local/go"
GOSUMDB="sum.golang.google.cn"
```

##### GO111MODULE
Go语言提供了 GO111MODULE 这个环境变量来作为 Go modules 的开关，其允许设置以下参数：

auto：只要项目包含了 go.mod 文件的话启用 Go modules，目前在 Go1.11 至 Go1.14 是默认值。
on：启用 Go modules，推荐设置，将会是未来版本中的默认值。
off：禁用 Go modules，不推荐设置。

##### GOPROXY
这个环境变量主要是用于设置 Go 模块代理（Go module proxy），其作用是用于使 Go 在后续拉取模块版本时能够脱离传统的 VCS 方式，直接通过镜像站点来快速拉取。

GOPROXY 的默认值是：https://proxy.golang.org,direct，这有一个很严重的问题，就是 proxy.golang.org 在国内是无法访问的，因此这会直接卡住你的第一步，所以你必须在开启 Go modules 的时，同时设置国内的 Go 模块代理，执行如下命令：
`go env -w GOPROXY=https://goproxy.cn,direct`

GOPROXY的值是一个以英文逗号 “,” 分割的 Go 模块代理列表，允许设置多个模块代理，假设你不想使用，也可以将其设置为 “off” ，这将会禁止 Go 在后续操作中使用任何 Go 模块代理。

###### direct是什么
而在刚刚设置的值中，我们可以发现值列表中有 “direct” 标识，它又有什么作用呢？

实际上 “direct” 是一个特殊指示符，用于指示 Go 回源到模块版本的源地址去抓取（比如 GitHub 等），场景如下：当值列表中上一个 Go 模块代理返回 404 或 410 错误时，Go 自动尝试列表中的下一个，遇见 “direct” 时回源，也就是回到源地址去抓取，而遇见 EOF 时终止并抛出类似 “invalid version: unknown revision…” 的错误。

#### 开启GO Modules
```
➜  ~ go env -w GO111MODULE=on
warning: go env -w GO111MODULE=... does not override conflicting OS environment variable
➜  ~ export GO111MODULE=on   
➜  ~ go env                  
GO111MODULE="on"
```

如果对应的系统环境变量有值了（进行过设置），会出现如下警告信息：warning: go env -w GO111MODULE=... does not override conflicting OS environment variable。

可以通过直接设置系统环境变量（写入对应的.bash_profile文件亦可）来实现这个目的：

`➜  ~ export GO111MODULE=on`

##### 关于环境变量优先级的问题
go env -w的优先级是最低的(相当于默认配置)，然后系统环境变量次之(环境变量配置在本地，例如配置.zshrc/.bashrc里面的)，当前shell的优先级最高(当前shell export的最高，关闭当前shell,配置的就失效了)

因此如果想设置一次性环境变量，直接在当前终端export就可以了，关闭当前shell后，配置就失效了，所以可以不同的shell终端可以拥有不同的环境变量.

如果GO11MODULES=off,需要下载翻墙包，那要怎么办呢，可以通过如下命令
`GO11MOGULE=on go get golang.org/x/text@v0.3.2`

#### GO MOD命令
| 命令          | 作用                           | 说明                                   |
| --------------- | -------------------------------- | ---------------------------------------- |
| go mod init     | 生成 go.mod 文件             | go mod初始化                          |
| go mod download | 下载 go.mod 文件中指明的所有依赖 | 一般配合vendor使用                 |
| go mod tidy     | 整理现有的依赖            | 有点鸡肋                             |
| go mod graph    | 查看现有的依赖结构      | 有点鸡肋，依赖冲突可能有用，但是不好排查 |
| go mod edit     | 编辑 go.mod 文件             | 有点鸡肋，一般直接在编辑器里修改 |
| go mod vendor   | 导出项目所有的依赖到vendor目录 | 一般配合download使用               |
| go mod verify   | 校验一个模块是否被篡改过 |                                          |
| go mod why      | 查看为什么需要依赖某模块 |                                          |


### Go Modules下的go get行为

开启go modules后，如何导入数据包呢，答案是跟没开启一样，直接通过 go get导入数据包

#### go get 命令

常用的拉取命令如下:
| 命令           | 作用                                                           | 说明         |
| ---------------- | ---------------------------------------------------------------- | -------------- |
| go get           | 拉取依赖，会进行指定性拉取（更新），并不会更新所依赖的其它模块。 | 获取包      |
| go get -u        | 更新现有的依赖，会强制更新它所依赖的其它全部模块，不包括自身。 | 获取最新包 |
| go get -u -v     | 更新现有的依赖, 并输出详细更新信息。            | 查看详情   |
| go get -u -t ./… | 更新所有直接依赖和间接依赖的模块版本，包括单元测试中用到的。 | 可以获取全部包 |

#### go get 获取指定版本
我想选择具体版本应当如何执行呢，如下:
| 命令                          | 作用                                               |
| ------------------------------- | ---------------------------------------------------- |
| go get golang.org/x/text@latest | 拉取最新的版本，若存在tag，则优先使用。 |
| go get golang.org/x/text@master | 拉取 master 分支的最新 commit。              |
| go get golang.org/x/text@v0.3.2 | 拉取 tag 为 v0.3.2 的 commit。                  |
| go get golang.org/x/text@342b2e | 拉取 hash 为 342b231 的 commit，最终会被转换为 v0.3. |


#### go get 版本选择
go get拉取的时候有两种选择:
1. 所拉取的模块带有发布tags:
- 如果只有单个模块，那么就取主版本号最大的那个tag
- 如果有多个模块，则推算相应的模块路径，取主版本号最大的那个tag（子模块的tag的模块路径会有前缀要求）

2. 所拉取的模块没有发布tags:
- 默认取主分支最新一次 commit 的 commithash。

### go mod 依赖问题

#### go mod 版本格式
go mod的版本信息如下：
![go mod版本信息](https://image.eddycjy.com/6e556b628df36b1fd3800fb9d91a0d16.jpg)

其版本格式为“主版本号.次版本号.修订号”，版本号的递增规则如下：

- 主版本号：当你做了不兼容的 API 修改。
- 次版本号：当你做了向下兼容的功能性新增。
- 修订号：当你做了向下兼容的问题修正。

假设先行版本号或特殊情况，可以将版本信息追加到“主版本号.次版本号.修订号”的后面，作为延伸，如下:
![go mod版本信息](https://image.eddycjy.com/b45438512cbb44015402da1a98190ac0.jpg)

#### go mod 依赖
现在我们已经有一个模块，也有发布的 tag，但是一个模块往往依赖着许多其它许许多多的模块，并且不同的模块在依赖时很有可能会出现依赖同一个模块的不同版本，如下图（来自Russ Cox）：
![go mod版本信息](https://image.eddycjy.com/7d509e8945fa31b7986369986c58e6f4.jpg)

在上述依赖中，模块 A 依赖了模块 B 和模块 C，而模块 B 依赖了模块 D，模块 C 依赖了模块 D 和 F，模块 D 又依赖了模块 E，而且同模块的不同版本还依赖了对应模块的不同版本。那么这个时候 Go modules 怎么选择版本，选择的是哪一个版本呢？

我们根据 proposal 可得知，Go modules 会把每个模块的依赖版本清单都整理出来，最终得到一个构建清单，如下图（来自Russ Cox）：
![go mod版本信息](https://image.eddycjy.com/2bd0bed89d9300c0aac24c7bc72a6307.jpg)

我们看到 rough list 和 final list，两者的区别在于重复引用的模块 D（v1.3、v1.4），其最终清单选用了模块 D 的 v1.4 版本，主要原因：

语义化版本的控制：因为模块 D 的 v1.3 和 v1.4 版本变更，都属于次版本号的变更，而在语义化版本的约束下，v1.4 必须是要向下兼容 v1.3 版本，因此认为不存在破坏性变更，也就是兼容的。

模块导入路径的规范：主版本号不同，模块的导入路径不一样，因此若出现不兼容的情况，其主版本号会改变，模块的导入路径自然也就改变了，因此不会与第一点的基础相冲突。

#### go mod 引来引发的问题
在导入模块的时候，社区规范的愿景很好，但是每个开发者不一定会严格遵循社区规范开发，最后的结果就是导入依赖的时候会出现不兼容的问题，那要如何解决呢？
1. 获取依赖图， go mod graph | grep "依赖", 例如: go mod graph | grep "submariner"
2. 使用replace替换为你你需要的版本，例如: github.com/submariner-io/submariner => github.com/submariner-io/submariner v0.2.0

### 参考文档
- https://colobu.com/2019/09/23/review-go-module-again/ //鸟窝
- https://eddycjy.com/posts/go/go-moduels/2020-02-28-go-modules/   //煎鱼(非常详细，推荐阅读)
- https://qcrao.com/2020/04/27/accident/ //码农桃花源(饶大还是幽默的)
- https://mp.weixin.qq.com/s/ENKV234bolS8UWd73JtRnQ