---
title:  "go实战读书笔记（十三）: channel"
date: 2018-04-15T16:12:23+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["Go In Action"]
---

中文书里把channel翻译成通道, 虽然很形象, 还是不喜欢, 就保持英文名channel吧.

在上一篇介绍了race condition, 以及通过使用mutex或者atomic来解决race condition问题. 

但atomic或者mutex并没有使得编写并发程序更简单, 甚至可以说, 这是其它主流语言类似C++解决race condition的方式.

在go里, 可以使用channel发送和接收需要的共享资源, 在goroutine之间做同步.

</br>

# channel的声明
---

声明channel时, 需要指定共享数据类型, 这些类型可以是内置类型, 命名类型, 结构类型, 引用类型或者是指针.

go是通过make来创建一个channel. channel 分为两种, **无缓冲channel**, 和**有缓冲channel**.

```go
// 定义一个无缓冲的整型channel.
unbuffered := make(chan int)

// 有缓冲的字符串channel.
buffered := make(chan string, 10)
```
有缓冲跟无缓冲的channel区别就是在创建的时候有没有指定缓冲区大小. 

创建channel时, 是否使用缓冲, 可以起到很多不同作用, 也是可以帮助利用goroutine建立很多不同的编程模型.

channel写入&读取
```go
buffered <- "Gopher"

value := <- buffered
```

</br>

# 无缓冲 channel
---
无缓冲channel是指在接收前没有能力保存任何值的channel. 这种类型的channel要求发送goroutine和接收goroutine同时准备好, 才能**持续完成** 发送和接收的操作.  无缓冲channel故而又被成为`同步channel`.

To summarize 无缓冲channel的几种case:

  * 发送方已经往无缓冲channel里写入, 接收方未接收, 那么此时无缓冲channel会block 发送方继续往里写.
  * 发送方已经往无缓冲channel里写入, 接收方未接收, 发送方可以接续写入.
  * 发送方尚未往无缓冲channel写入, 接收方尝试读取会失败, 此时接收方会阻塞在读取channel信息代码块.

来个简单的例子, 前面一节是通过waitgroup来防止因为main函数退出而导致其它定义的goroutine没有执行完毕就结束. 有同步管道, 可以更简洁.
示例代码:https://play.golang.org/p/W7xclyvnWdh

```go
package main

import (
	"fmt"
)

func client(ch chan string) {
	ch <- "hello"
}

func main() {
	var ch chan string
	ch = make(chan string)
	go client(ch)

	fmt.Println("client says:", <-ch)
}

```
```
// output
client says: hello
```
代码很简单, 主函数等待函数client发送信息, 收到信息之后从channel内读取client发送的信息, 打印出来才退出.

</br>

# 有缓冲channel
---
有缓冲的channel是一种能在被接收前储存一个或多个值的channel. 这种类型channel并不要求goroutine之间必须同时完成发送和接收. 作为接收方, 只要在channel中没有值时, 才会被堵塞. 作为发送方, 只有channel中没有可用缓冲区容纳发送值时, 发送才会堵塞.

稍稍修改下上文无缓冲channel例子:https://play.golang.org/p/NQsWMgb66MC
```go
package main

import (
	"fmt"
)

func client(ch chan string) {
	ch <- "hello, "
	ch <- "world"
	ch <- "I am client"
	fmt.Println("一口气往channel里写了三个数据, 感觉自己萌萌哒.")
}

func main() {
	var ch chan string
	ch = make(chan string, 3)
	go client(ch)

	fmt.Println("client says:", <-ch)
	fmt.Println("client says:", <-ch)
	fmt.Println("client says:", <-ch)
}
```
输出
```go
一口气往channel里写了三个数据, 感觉自己萌萌哒.
client says: hello, 
client says: world
client says: I am client
```
</br>

关于有缓冲channel, 这里要引起注意的是, 一旦容量声明, go 编译器在这里会给channel预分配内存, 也就是说如果你定义了一个有缓冲channel, 如下代码, 缓冲区size很大, (1M), 那么你程序启动的时候这部分内存也会被预先占用. 

```go
package main

import (
	"time"
)

// main is the entry point for all Go programs.
func main() {
	ch := make(chan string, 1000000)
	time.Sleep(100 * time.Second)
}
```
程序刚启动, 就这么简单的两行代码, 占用了 **16.3M** 内存, 所以使用有缓冲区的时候, 一定要注意size的设定. 

如果把有缓冲区channel类比为数组的话, 那go里现在还没有类似切片可以动态增长channelsize的channel结构, 这也是当前被人诟病的一点.

我之前有照着这篇wiki: :https://github.com/npat-efault/musings/wiki/Elastic-channels 利用slice实现过可动态增加size的有缓冲channel, 性能和效果都还可以, 有兴趣的小伙伴也可以尝试一下. 

</br>

# 单向channel
---

单向channel用来修饰channel类型的变量或者参数, 表示只能从该channel接受值, 或者只能往该channel发送值.

* 发送值的channel类型, 表示只能向channel发送数据 `chan<- T`
* 接收值的channel类型, 表示只能从channel接收数据 `<-chan T`

比如os/signal包中的noitfy 函数, 表示notify只会往channel里发送数据:

```go
func Notify(c chan<- os.Signal, sig ...os.Signal) {
    ...
}
```

</br>

---

附上第6章小结:

* 并发是指goroutine运行时是相互独立的.
* 使用关键词go创建goroutine来运行函数.
* goroutine在逻辑处理器上执行, 逻辑处理器有独立的系统线程和运行队列.
* 竞争状态是指两个或者多个goroutine试图访问同一个资源.
* 原子函数和互斥锁提供了一种防止出现race condition的办法.
* channel提供了一种在两个goroutine之间共享数据的简单方法.
* 无缓冲channel保障同时交换数据, 而有缓冲channel不保证这一点.