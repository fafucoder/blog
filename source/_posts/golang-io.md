---

title: golang io.Reader/io.Writer
date: 2021-02-21 14:14:24
tags:
- golang
categories:
- golang
---

### 概述

在使用Go语言的过程中，无论是实现web应用程序，还是控制台输入输出，又或者是网络操作，不可避免的会遇到IO操作，使用到io.Reader和io.Writer接口。

![io](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gnv6vcq9iej315m078jvu.jpg)

### io.Reader

`io.Reader` 表示一个读取器，它将数据从某个资源读取到传输缓冲区。在缓冲区中，数据可以被流式传输和使用。

![io reader](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gnv6wqpi95j31720a2n1s.jpg)

io.Reader接口定义如下：

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

`io.Reader`接口定义了`Read(p []byte) (n int, err error)`方法，我们可以使用它从Reader中读取一批数据。在Reader中：

- 一次最多读取`len(p)`长度的数据
- 读取遭遇到error(io.EOF或者其它错误), 会返回已读取的数据的字节数和error
- 即使读取字节数< len(p),也不会更改p的大小
- 当输入流结束时，调用它可能返回 `err == EOF` 或者 `err == nil`，并且`n >=0`, 但是下一次调用肯定返回 `n=0`, `err=io.EOF`

### io.Writer

`io.Writer` 表示一个编写器，它从缓冲区读取数据，并将数据写入目标资源。

![io writer](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gnv73ri3h0j315o0ai435.jpg)

io.Writer接口定义如下：

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

`Write()` 方法有两个返回值，一个是写入到目标资源的字节数，一个是发生错误时的错误。

### 接口继承关系

围绕着`io.Reader io.Writer `接口的实现，主要有：

- net.Conn, os.Stdin, os.File: 网络、标准输入输出、文件的流读取

- strings.Reader: 把字符串抽象成Reader

- bytes.Reader: 把`[]byte`抽象成Reader

- bytes.Buffer: 把`[]byte`抽象成Reader和Writer

- bufio.Reader/Writer: 抽象成带缓冲的流读取（比如按行读写）

![接口继承关系](https://fafucoder-1252756369.cos.ap-nanjing.myqcloud.com/008eGmZEly1gnv6fu1e37j312k0l4gsz.jpg)



```go
func stringReader() {
	reader := strings.NewReader("hello world")
	b := make([]byte, 4)

	for {
		n, err := reader.Read(b)
		if err != nil {
			if err == io.EOF {
				fmt.Println("finish read")
				break
			}

			fmt.Println(err)
			os.Exit(1)
		}
		fmt.Println(n, string(b[:n]))
	}
}

func bufferWriter() {
	providers := []string{
		"hello",
		"world",
		"golang",
		"is great",
	}

	var writer bytes.Buffer
	for _, s := range providers {
		n, err := writer.Write([]byte(s))
		if err != nil {
			fmt.Println(err)
			os.Exit(1)
		}

		if n != len(s) {
			fmt.Println("failed to write data")
			os.Exit(1)
		}
	}

	fmt.Println(writer.String())
}

func fileWriter() {
	file, err := os.Create("./files.txt")
	if err != nil {
		fmt.Println(err)
		os.Exit(1)
	}

	defer file.Close()

	providers := []string{
		"hello",
		"world",
		"golang",
		"is great",
	}

	for _, p := range providers {
		c, err := file.Write([]byte(p))
		if err != nil {
			fmt.Println(err)
			os.Exit(1)
		}

		if c != len(p) {
			fmt.Println("failed to write data")
			os.Exit(1)
		}
	}
}

//file reader
func fileReader() {
	file, err := os.Open("./files.txt")
	if err != nil {
		fmt.Println(err)
		os.Exit(1)
	}

	defer file.Close()

	p := make([]byte, 4)
	for {
		n, err := file.Read(p)
		if err != nil {
			if err == io.EOF {
				break
			}

			fmt.Println(err)
		}

		fmt.Println(string(p[:n]))
	}
}

//buffio
func bufReader() {
	file, err := os.Open("./files.txt")
	if err != nil {
		panic(err)
	}

	defer file.Close()

	p := make([]byte, 4)
	reader := bufio.NewReader(file)
	for {
		n, err := reader.Read(p)
		if err != nil {
			if err == io.EOF {
				break
			}
			panic(err)
		}

		fmt.Println(string(p[:n]))
	}
}

//ioutil 包
func ioutilReader() {
	bytes, err := ioutil.ReadFile("./files.txt")
	if err != nil {
		panic(err)
	}

	fmt.Println(string(bytes))
}
```

### 参考文档

- [Go中io包的使用方法](https://segmentfault.com/a/1190000015591319)
- [Go编程技巧--io.Reader/Writer](https://www.jianshu.com/p/758c4e2b4ab8)
- [从 io.Reader 中读数据](https://colobu.com/2019/02/18/read-data-from-net-Conn/)