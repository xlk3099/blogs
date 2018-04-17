---
title: "Go语言实战读书笔记（十六）: 标准库 - Log"
date: 2018-04-17T20:17:20+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["Go In Action"]
---

go语言提供了很强大的标准库, 其中经常被用到的就是log, json 跟 io. 这篇我们看下log的使用.

log在go语言开发中被使用的很多, 无论是用来debug程序, 还是用来帮助添加日志分析程序运行健康与否, 都基本离不开它.
关于log的起源, 其实是来自linux系统日志, 随后逐渐各大应用程序都有了自己的日志系统, 便于开发, 维护人员尽早发现程序问题. 

 现阶段, 比较容易接受的log信息是这样的:

> [Severity]-[TimeStamp]-[Application ID]-[ThreadID]-[Call Depth]-[MSG]

其中, [Applicatoin ID] 跟 [Thread ID] 在并行或是并发环境中很有必要, 在分布式系统上如果有多个版本应用程序运行, 可能还要加上[Application Version] 等额外信息方便迅速找到问题. 但最最基本的
[Severity] [TimeStamp] [Msg] 肯定是要有的.

`msg` 一般而言是string格式, 但好多主流开源软件也支持json输出格式, 或者[KV] pair输出格式.
比如go社区比较出名的[`logrus`](https://github.com/sirupsen/logrus) 

关于severity, 根据[RFC5424](https://tools.ietf.org/html/rfc5424#section-6.3.5) 对于系统日志, 有8个级别, 但对于应用程序, 最常用的是六个级别, 按严重程度从低到高排序: Trace, Info, Debug, Warn, Error, Fatal.

那么, 我们看看如何使用go自带的log包 实现我们自己的log库, 我们希望输出符合:

> [Severity]-[TimeStamp]-[Call Depth]-[Msg]


# log 包基本使用
---

一个典型的跟踪日志输出, 由log包产生:
```go
package main

import (
	"log"
)

func main() {
	log.Println("log 使用demo")
}
/// output
2018/04/17 20:25:18 log 使用demo
```
上述代码可以看到, 由log包产生的日志包含了, 日期-时间, 源文件代码所在行, 最后是日志消息.
用法看上去跟`fmt.Println`很相似, 区别是从`fmt.Println`变成了`log.Println`, 而且输出前添加了时间信息而已.
但这跟标准的log信息有些出入. 

好在go的log包邮很多参数可以设置, 让我们看下log的支持参数设置跟方法.

```go
const (
	Ldate         = 1 << iota     // 设置开启日志前缀日期: 2009/01/23
	Ltime                         // 设置开启日志前缀时间: 01:23:23
	Lmicroseconds                 // 设置开启日志前缀时间毫秒级: 01:23:23.123123.
	Llongfile                     // 设置开启日志前缀完整文件路径跟行数: /a/b/c/d.go:23
	Lshortfile                    // 设置开启日志前缀最终文件名跟行数: d.go:23. 会覆盖Llongfile设置
	LUTC                          // 如果Ldate活着Ltime被开启, 那么采用UTC格式
	LstdFlags     = Ldate | Ltime // 初始flags设置, 开启时间日期.
)
```

1 << iota 是go给参数设置值的一种常见方法, 这里就不介绍了, 感兴趣的小伙伴可以自己搜索下.
上面的设置其实就是:
```go
const (
	Ldate         = 1
	Ltime         = 2
	Lmicroseconds = 4
	Llongfile     = 8
	Lshortfile    = 16
	LUTC          = 32
	LstdFlags     = Ldate | Ltime
)
```
log包里提供了方法`SetFlags` 来开关这些设置.
还是最初的例子, 设置日志输出按照 `日期 时间精准到毫秒 长文件行数 信息` 格式:

```go
package main

import (
	"log"
)

func init() {
	log.SetFlags(log.Ldate | log.Ltime | log.Lmicroseconds | log.Llongfile)
}

func main() {
	log.Println("log 使用demo")
}
```

得到输出:
```
➜  xlk3099 go run main.go
2018/04/17 21:02:34.081423 /Users/qianqian/go/src/github.com/xlk3099/main.go:12: log 使用demo
```
在上面, 我们添加了一个新的函数`init`, 在里面我们设置`log.SetFlags(log.Ldate | log.Ltime | log.Lmicroseconds | log.Llongfile)`. 一般而言, 配置信息都会放在每个`init`函数里, 是go编程里的一个通用习惯.

上面就是log包自带的配置方式, 采用的是常见的通过bitwise `OR` operation, 来实现多个参数同时支持.

离我们文章一开始要求只差一步, 就是`Level` 信息. 

Log包有很多方法, 其中之一就是设置SetPrefix.
修改之前的init函数:
```go
func init() {
	log.SetFlags(log.Ldate | log.Ltime | log.Lmicroseconds | log.Llongfile)
	log.SetPrefix("[Trace]")
}
```
再运行main, 输出变成了:
```go
[Trace] 2018/04/17 21:48:49.646283 /Users/qianqian/go/src/github.com/xlk3099/main.go:13: log 使用demo
```

方便时方便, 但这只是一个Trace级别的输出, 很多情况下我们需要WARN, ERRO, 甚至是FATAL的输出, 不可能每次输出前都SetPrefix.

再仔细看下log包, 发现log包支持构造自己的Logger类型
```go
type Logger struct {
	mu     sync.Mutex // ensures atomic writes; protects the following fields
	prefix string     // prefix to write at beginning of each line
	flag   int        // properties
	out    io.Writer  // destination for output
	buf    []byte     // for accumulating text to write
}
```

先看下Logger的结构类型, 有四个字段:

* mu 互斥锁, 保证原子级别的写操作, 并保护多goroutine下prefix, flag, out, buf.
* prefix 用户自定义前缀
* flag 日志配置设定
* out 输出目的地
* buf 日志文本缓冲, 用于最后输出

而我们上面调用的`.SetPrefix`, `Println` 其实是用的log包默认的Logger `std`. 
到这里就比较简单了.

为了满足支持不同Level的log, 我们可以声明6个不同级别的logger, 然后在init里初始化它们.
```go
// 定义不同级别的logger
var (
	Trace *log.Logger
	Debug *log.Logger
	Info  *log.Logger
	Warn  *log.Logger
	Error *log.Logger
	Fatal *log.Logger
)

func init() {
	Trace = log.New(os.Stdout, "[Trace]: ", log.Ldate|log.Ltime|log.Lmicroseconds|log.Lshortfile)
	Debug = log.New(os.Stdout, "[Debug]: ", log.Ldate|log.Ltime|log.Lmicroseconds|log.Lshortfile)
	Info = log.New(os.Stdout, "[Info]: ", log.Ldate|log.Ltime|log.Lmicroseconds|log.Lshortfile)
	Warn = log.New(os.Stdout, "[Warn]: ", log.Ldate|log.Ltime|log.Lmicroseconds|log.Lshortfile)
	Error = log.New(os.Stdout, "[Error]: ", log.Ldate|log.Ltime|log.Lmicroseconds|log.Lshortfile)
	Fatal = log.New(os.Stdout, "[Fatal]: ", log.Ldate|log.Ltime|log.Lmicroseconds|log.Lshortfile)
}
```

测试:
```go
func main() {
	Trace.Println("这是Trace")
	Debug.Println("这是Debug")
	Info.Println("这是Info")
	Warn.Println("这是Warn")
	Error.Println("这是Error")
	Fatal.Println("这是Fatal")
}
```
输出:
```
> xlk3099 go run main.go
[Trace]: 2018/04/17 21:59:40.679369 main.go:28: 这是Trace
[Debug]: 2018/04/17 21:59:40.679435 main.go:29: 这是Debug
[Info]: 2018/04/17 21:59:40.679453 main.go:30: 这是Info
[Warn]: 2018/04/17 21:59:40.679457 main.go:31: 这是Warn
[Error]: 2018/04/17 21:59:40.679460 main.go:32: 这是Error
[Fatal]: 2018/04/17 21:59:40.679475 main.go:33: 这是Fatal
```

这样就完成了一个简单版的支持不同级别的logger. 


# logger 输出原理
---

再细看log包, 观察logger的工厂函数.
```go
func New(out io.Writer, prefix string, flag int) *Logger {
	return &Logger{out: out, prefix: prefix, flag: flag}
}
```
需要的三个参数都是Logger的内置字段. 注意到out是个io.Writter类型, 也就是说, logger的输出其实是不止stdout, 也可以是file(文件), 凡是实现了io.Writer接口Write的类型都可以. 最后看一下Output函数,

```go
func (l *Logger) Output(calldepth int, s string) error {
	now := time.Now() // get this early.
	var file string
	var line int
	l.mu.Lock()
	defer l.mu.Unlock()
	if l.flag&(Lshortfile|Llongfile) != 0 {
		// Release lock while getting caller info - it's expensive.
		l.mu.Unlock()
		var ok bool
		_, file, line, ok = runtime.Caller(calldepth)
		if !ok {
			file = "???"
			line = 0
		}
		l.mu.Lock()
	}
	l.buf = l.buf[:0]
	l.formatHeader(&l.buf, now, file, line)
	l.buf = append(l.buf, s...)
	if len(s) == 0 || s[len(s)-1] != '\n' {
		l.buf = append(l.buf, '\n')
	}
	_, err := l.out.Write(l.buf)
	return err
}
```

其实实现原理很简单, 就是将所要求的信息, 按照定义格式写入到logger的 buf类别里, 最后写入到out.

到这里logger就讲完了. 有兴趣的小伙伴可以在这个基础上尝试添加更多很多logger格式:

* 给stdout(命令行)输出的log 添加颜色, (仅限linux or mac系统)
* 给输出格式化, 用成kv模式.
* 用一个logger实现Trace, Debug, Info, Warn, Error, Fatal的不同级别输出.
* 可以尝试log输出到文件, 邮件, git 甚至是slack里.

