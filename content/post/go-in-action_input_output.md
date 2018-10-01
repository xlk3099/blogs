---
title: "go实战读书笔记（十九）: 标准库 - Input & Output"
date: 2018-04-19T21:36:30+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["Go In Action"]
---

unix 系统(包括linux&macos) 一个很好用的地方就是一个程序的输出可以作为另一个程序的输入. 把多个不同作用的小程序整合到一起, 写个script, 可以做到很神奇的事. 这些程序, 使用stdout跟stdin, 在进程之间传递数据.

对应stdout跟stdin, go的io包提供了io.Writer和io.Reader两个接口, 任何数据类型实现这两个接口, 就可以使用io包里提供的所有功能.

# Reader and Writer 接口

---

先看一下Writer接口定义:

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

Writer接口只有一个函数`Write`. 从p里向底层的数据流写入len(p)字节的数据, 返回 `0<=n<=len(p)`的一个整型数值表示写入了多少长度的byte, 跟error类型. 如果没能彻底写完len(p), 那必须返回一个非nil的error. 这里要注意的是, 在write函数里, byte切片不能被修改, 临时也不行.

在看下Readr接口:

```go
type Reader interface {
    Read(p []byte) (n int, err, error)
}
```

关于如何准确实现Reader接口, 官方代码提供了4个准则:

1. 跟Write函数对应, Read函数会尝试读取长度len(p)的byte到p里, 它会返还读取的长度跟error. 跟Write不同的时, 及时Read 返回的n小于len(p), error也可能是nil.
2. 这里要注意的是, read可能返回的错误类型EOF, 表示stream读取结束. 
3. 如果Read返回错误, Read的调用函数也应该优先处理写入到p的n个byte, 这样可以正确地处理再读取了n个byte之后发生的io错误, 包括EOF错误.
4. 实现Read函数的时候不推荐返回n==0同时err==nil, 而且调用者也应该将这种情况视为Read函数什么都没做, 也不能将其看成读取结束.

# 举个🌰
---

我们都知道在go里面, 可以使用`fmt.Println` 或者`fmt.Printf` 将信息输出到命令行, 其实命令行对应的输出地是`stdout`.

```go
package main

import (
    "bytes"
    "fmt"
    "os"
)

func main() {
    var b bytes.Buffer
    b.Write([]byte("Hello "))

    fmt.Fprintf(&b, "World!")

    b.WriteTo(os.Stdout)
}

// 输出
Hello World!
```
之前我们已经接触过`fmt.Fprintf`了, 这里再强化一遍,
```go
func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error) {
    p := newPrinter()
	p.doPrintf(format, a)
	n, err = w.Write(p.buf)
	p.free()
	return
}
```

函数`Fprintf`第一个接受的参数必须是实现了io.Writer的数据类型. 而bytes包里的Buffer也确实实现了Write函数,

```go
// Write 将p里面的内容追加到b.buf里, 并增加buf的size如果超出了buf的cap. 
// 这里要注意的是, 如果buf的size太大, 会返回ErrTooLarge的错误.
func (b *Buffer) Write(p []byte) (n int, err error) {
	b.lastRead = opInvalid
	m, ok := b.tryGrowByReslice(len(p))
	if !ok {
		m = b.grow(len(p))
	}
	return copy(b.buf[m:], p), nil
}
```

所以`Fprintf` 其实就是讲`World!` 添加到b已有的[]byte里.

在函数最后一行, 使用WriteTo将Buffer类型变量b的[]byte写到os.Stdout里, 好吧, 先看下WriteTo函数:

```go
// WriteTo writes data to w until the buffer is drained or an error occurs.
// The return value n is the number of bytes written; it always fits into an
// int, but it is int64 to match the io.WriterTo interface. Any error
// encountered during the write is also returned.
func (b *Buffer) WriteTo(w io.Writer) (n int64, err error) {
	b.lastRead = opInvalid
	if nBytes := b.Len(); nBytes > 0 {
		m, e := w.Write(b.buf[b.off:])
		if m > nBytes {
			panic("bytes.Buffer.WriteTo: invalid Write count")
		}
		b.off += m
		n = int64(m)
		if e != nil {
			return n, e
		}
		// all bytes should have been written, by definition of
		// Write method in io.Writer
		if m != nBytes {
			return n, io.ErrShortWrite
		}
	}
	// Buffer is now empty; reset.
	b.Reset()
    return n, nil
}
```

WriteTo函数接受一个`io.Writer`again... 再看下os包关于Stdout的定义.

```go
var (
    Stdin  = NewFile(uintptr(syscall.Stdin), "/dev/stdin")
    Stdout = NewFile(uintptr(syscall.Stdout), "/dev/stdout")
    Stderr = NewFile(uintptr(syscall.Stderr), "/dev/stderr")
)
```

这里Stdin, Stdout, Stderr 分别指向了unix系统下的`标准输入流`, `标准输出流`, `标准错误流` 三个模式(文件). 这三种模式都是File类型, *File类型是实现了io.Writer的.

# curl的实现
---

熟悉unix系统的小伙伴对curl肯定不陌生, 这个指令可以对指定url发起http get, 并保存其返回内容. 通过http, io, 跟os包, 我们可以用go实现自己的curl.

```go
package main

import (
	"io"
	"log"
	"net/http"
	"os"
)

func main() {
	// 读取用户输入url发出请求, 并将结果保存在r里.
	r, err := http.Get(os.Args[1])
	if err != nil {
        // 请求返回错误, 打印错误
		log.Fatalln(err)
	}

	// 读取用户第二个输入作为curl返回内容保存文件.
	file, err := os.Create(os.Args[2])
	if err != nil {
		log.Fatalln(err)
	}
	defer file.Close()

	// 设立multiwriter, 输出指向stdout跟指定文件
	dest := io.MultiWriter(os.Stdout, file)

	// 读取返回结果, 并将结果保存到dest里
	io.Copy(dest, r.Body)
	if err := r.Body.Close(); err != nil {
		log.Println(err)
	}
}
```

这个例子demo了一个curl的基本骨架, 将其build成app

> go build -o app -i main.go

基本用法:
 
>  app&emsp;[url]&emsp;[output]

e.g:
```

> app https://www.google.com test.out
```

io包支持流的方式处理数据, io包的大部分操作和类型都是基于Reader跟Writer这两个接口. 类型类似 http数据流, 文件流等, 也可以用io的Reader跟Writer来表示.

---
附第8章总结:

* 标准库有特殊的保证,并且被社区广泛应用。 
* 使用标准库的包会让你的代码更易于管理,别人也会更信任你的代码。 
* 100 余个包被合理组织,分布在 38 个类别里。
* 标准库里的 log 包拥有记录日志所需的一切功能。
* 标准库里的 xml 和 json 包让处理这两种数据格式变得很简单。 
* io 包支持以流的方式高效处理数据。 
* 接口允许你的代码组合已有的功能。
* 阅读标准库的代码是熟悉 go习惯的好方法。

