---
title: "go实战读书笔记（三）：go tools 开发工具"
date: 2018-04-09T22:38:33+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["Go In Action"]
---
go 本身是一个非常强大的命令工具.

打开命令行, 输入`go`, 敲击回车:
```
➜  ~ go
Go is a tool for managing Go source code.

Usage:

	go command [arguments]

The commands are:

	build       compile packages and dependencies
	clean       remove object files and cached files
	doc         show documentation for package or symbol
	env         print Go environment information
	bug         start a bug report
	fix         update packages to use new APIs
	fmt         gofmt (reformat) package sources
	generate    generate Go files by processing source
	get         download and install packages and dependencies
	install     compile and install packages and dependencies
	list        list packages
	run         compile and run Go program
	test        test packages
	tool        run specified go tool
	version     print Go version
	vet         report likely mistakes in packages

Use "go help [command]" for more information about a command.

Additional help topics:

	c           calling between Go and C
	buildmode   build modes
	cache       build and test caching
	filetype    file types
	gopath      GOPATH environment variable
	environment environment variables
	importpath  import path syntax
	packages    package lists
	testflag    testing flags
	testfunc    testing functions

Use "go help [topic]" for more information about that topic.
```
我们可以看到go 提供了很多指令集合,`build`,`clean`,`doc`,`env`,`bug` , `test` 等等.

在这篇文章里,我们会cover go几个最基本的指令.

# go vet
---
`go vet` 是一个非常实用的指令, 它可以用来帮助检测代码中的常见错误.

  1. Printf 类函数调用时,类型匹配错误的参数. 
  2. 定义常用方法时,方法签名的错误.
  3. 错误的结构标签.
  4. 没有指定字段名的结构字面量. >>> `讲真, what the hell is this?` <<<

书中的例子
```go
package main

import "fmt"

func main() {
	fmt.Printf("The quick brown fox, jumped over lazy dogs", 3.14)
}

```
对这段代码进行`go vet` 检测,
``` 
go vet main.go
```
我们得到:
```
./playground.go:6: Printf call has arguments but no formatting directives
```
当然更为方便的方法, 是在你的`code editor` 里面安装`go vet`的插件. 比如`vscode`, 安装完go插件后, 设置好`GOPATH`, 每次编辑结束, 保存代码, go插件会自动运行`go vet` 在代码里会高亮各种不符合规范的代码.

# go fmt
---
`fmt`是`format`的简称, `go fmt` 是go语言社区非常受欢迎的一个命令. `fmt` 会将开发人员的代码布局格式成跟go源码类似风格. 举个🌰:

```go
if err != nil { return err }
```

在运行`go fmt`之后, 代码会自动调整成:

```go
if err != nil {
	return err
}
```

跟`go vet`一样, 也可以在你的code editor 或者IDE里面安装`fmt`插件. 这样每次保存代码会自动进行format.

# go doc (or godoc)
---
开发软件时, 我们最常听到的一句吐槽就是: 找不到对应文档. go提供了两种非常方便的文档查看方式, 在命令行输入 `go help doc` 指令

```
~ go help doc
usage: go doc [-u] [-c] [package|[package.]symbol[.methodOrField]]
```

我们会发现, `go doc` 能接受的参数有`包`,`方法`,`结构体`.

  * 终端查看文档: 对于喜欢使用命令行开发程序的人员(vim),用终端查看文档会很方便. 举个🌰, 查看`archive/tar` 包的文档.
  ```
	➜  ~ go doc tar
	package tar // import "archive/tar"

	Package tar implements access to tar archives.

	Tape archives (tar) are a file format for storing a sequence of files that
	can be read and written in a streaming manner. This package aims to cover
	most variations of the format, including those produced by GNU and BSD tar
	tools.

	const TypeReg = '0' ...
	var ErrHeader = errors.New("archive/tar: invalid tar header") ...
	type Format int
		const FormatUnknown Format ...
	type Header struct{ ... }
		func FileInfoHeader(fi os.FileInfo, link string) (*Header, error)
	type Reader struct{ ... }
		func NewReader(r io.Reader) *Reader
	type Writer struct{ ... }
		func NewWriter(w io.Writer) *Writer
	```
  * 浏览器查看: 有时候我们需要跳转到其他文档查看相关细节.这时候用命令行就显得很麻烦.
    好在go同时提供了浏览器查看的方式. 打开终端,输入:
	```
	➜  ~ godoc -http=:6060 
	```
	这个指令会在端口`6060` 启动web服务器. 打开浏览器,导航到`http://localhost:6060` 我们
	就可以查看所有的go 标准库, 和`GOPATH`下所有的go源代码文档.

	{{% center %}}![image](https://user-images.githubusercontent.com/1768412/38507056-0cc3602e-3c4e-11e8-9fc6-de7eeafc449d.png "png"){{% /center %}}

go文档工具最棒的地方在于, 支持开发人员自己写的代码. 如果开发人员遵从简单的柜子来写代码, 比如大写的method 代表`export`, 需要添加注释等等, 这些代码的文档也会自动包含在godoc生成的文档里. 这个会在后面的章节里提到...

	>>>还是那句话,在你的IDE里面安装好go插件, 会自动提醒你文档规则.<<<

