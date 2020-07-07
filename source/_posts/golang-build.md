---
title: golang-build
date: 2020-07-07 12:24:32
tags:
- golang
categories:
- golang
---

### build 参数
使用方法`usage: go build [-o output] [-i] [build flags] [packages]`

go build的使用比较简洁，所有的参数都可以忽略，直到只有go build，这个时候意味着使用当前目录进行编译, 因此:

`go build = go build . = go build main.go`

go build 主要参数是 -o output 指定编译输出的名称，代替默认的包名。

此外还有一些基础参数，包括install, run, test都能用的参数包括:
```
-gcflags 'arg list': 垃圾回收参数

-ldflags '-s -w': 压缩编译后的体积
    -s: 去掉符号表
    -w: 去掉调试信息，不能gdb调试了

-p n
    开多少核cpu来并行编译，默认为本机CPU核数.

-work
    打印临时工作目录的名称，并在退出时不删除它

-race
    同时检测数据竞争状态，只支持 linux/amd64, freebsd/amd64, darwin/amd64 和 windows/amd64.

-tags 'tag list'
    构建出带tag的版本.

-linkshared(-buildmode=shared)
    链接到以前使用创建的共享库


-buildmode mode
    编译模式(go help buildmode)

```

其中通过利用 `-ldflags` 参数，可以向链接器传递指令。向链接器传一个 -X 指令可以设置程序中字符串变量的值。利用这个方法能够实现例如在编译时设置程序的版本信息， 例如：

```
package main
 
import "fmt"
import "flag"
 
var _version_ = "v0.1"
 
func main() {
    var version bool
    flag.BoolVar(&version, "v", false, "-v")
    flag.Parse()
 
    if version {
        fmt.Printf("Version: %s", _version_)
    }
}
```

通过设置 `go build -ldflags '-X main._version_="v0.2"'`编译后输出的版本信息就是v0.2, 通过-X参数可以很方便的设置一些频繁改动的变量。需要注意的是-X只能给string类型变量赋值

如果要设置多个变量可以: `go build -ldflags "-X importpath.name=value -X importpath_2.name_2=value_2"`

如果要赋值的变量包含空格，需要用引号将 -X 后面的变量和值都扩起来：`go build -ldflags "-X 'importpath.name=a string contains space' -X 'importpath_2.name_2=v`


通过利用`-gcflags`参数， 可以做一些逃逸分析等操作，用`-gcflags`给go编译器传入参数，也就是传给go tool compile的参数，因此可以用`go tool compile --help`查看所有gcflags可用的参数， 例如：
```
go build -gcflags -gcflags='log=-N -l' main.go

-N参数代表禁止优化, -l参数代表禁止内联,
go在编译目标程序的时候会嵌入运行时(runtime)的二进制,
禁止优化和内联可以让运行时(runtime)中的函数变得更容易调试(runtime还是需要好好研读的)
```

### Go 交叉编译
go build 提供了跨平台编译，默认情况下，都是根据我们当前的机器生成的可执行文件，比如你的是Linux 64位，就会生成Linux 64位下的可执行文件。

可以通过go env 来查看当前的环境变量

```
➜  ~ go env
GO111MODULE="on"
GOARCH="amd64"
GOBIN="/Users/dawn/go/bin"
GOCACHE="/Users/dawn/Library/Caches/go-build"
GOENV="/Users/dawn/Library/Application Support/go/env"
GOEXE=""
GOFLAGS=""
GOHOSTARCH="amd64"
GOHOSTOS="darwin"
GOINSECURE=""
GONOPROXY=""
GONOSUMDB=""
GOOS="darwin"
GOPATH="/Users/dawn/go"
GOPRIVATE=""
GOPROXY="https://goproxy.cn,direct"
GOROOT="/usr/local/Cellar/go/1.14.2_1/libexec"
GOSUMDB="sum.golang.google.cn"
GOTMPDIR=""
GOTOOLDIR="/usr/local/Cellar/go/1.14.2_1/libexec/pkg/tool/darwin_amd64"
GCCGO="gccgo"
AR="ar"
CC="clang"
CXX="clang++"
CGO_ENABLED="1"
GOMOD="/dev/null"
CGO_CFLAGS="-g -O2"
CGO_CPPFLAGS=""
CGO_CXXFLAGS="-g -O2"
CGO_FFLAGS="-g -O2"
CGO_LDFLAGS="-g -O2"
PKG_CONFIG="pkg-config"
GOGCCFLAGS="-fPIC -m64 -pthread -fno-caret-diagnostics -Qunused-arguments -fmessage-length=0 -fdebug-prefix-map=/var/folders/4g/tx2_phhn0gx5j81d1y286bf80000gn/T/go-build124067622=/tmp/go-build -gno-record-gcc-switches -fno-common"
```

在上面的环境变量中，与跨平台编译有关的变量有GOOS, GOARCH, GOOS指的是目标操作系统, GOARCH指的是目标处理器的架构。

GOOS可选的值包括: android，darwin（MACOS 10.11和上述和IOS),  dragonfly，freebsd，illumos，js， linux，netbsd，，openbsd 和。 plan9solariswindows

GOARCH可选的值包括 amd64（64位x86，最成熟的端口）， 386（32位x86），arm（32位ARM），arm64（64位ARM）， ppc64le（PowerPC 64位，little-endian），ppc64（PowerPC 64位（big-endian）， mips64le（MIPS 64位，little-endian），mips64（MIPS 64位，big-endian）， mipsle（MIPS 32位，little-endian），mips（MIPS 32位，big-endian） ）， s390x（IBM System z 64位，big-endian）和 wasm（WebAssembly 32位)。

关于他们的对应关系可以参考[官方文档](https://golang.org/doc/install/source#environment)

在交叉编译中通过指定不同的GOOS跟GOARCH，就可以编译成不同平台的可执行文件

例如可以在linux系统下编译成window跟mac的可执行文件
```
CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build main.go
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build main.go
```

可以在mac系统中编译成linux跟windows的可执行文件
```
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build main.go
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build main.go
```

### 参考文档
- https://cloud.tencent.com/developer/article/1410259
- https://www.cnblogs.com/Dong-Ge/articles/11276862.html
- https://blog.csdn.net/u013870094/article/details/78500478
- https://blog.csdn.net/zl1zl2zl3/article/details/83374131