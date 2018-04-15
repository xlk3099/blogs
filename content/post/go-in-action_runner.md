---
title: "Go语言实战读书笔记（十四）: 并发模式:Runner"
date: 2018-04-15T17:53:27+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["Go In Action"]
---

在前几章中, 学到了go语言的并发, 通道是如何工作的, 接下来会介绍三种go语言常见的并发模式, runner, pool, worker.

本篇会展示使用通道来监视程序的执行时间, 生命周期, 监听终止信号等等. 这个包把它称之为Runner, Runner在后台处理任务程序会很有用. 可以作为`cron task`, 也可以作为基于定时任务的云环境任务.

下面们定义了一个Runner类型, 从设计角度上思考, Runner需要完成下列工作:
  * 程序在分配时间内完成工作, 正常终止.
  * 程序没有完成工作, 应该被强制终止, 并返回超时错误.
  * 接收到系统发送的终端信号, 应该完成当前任务, 清理状态并停止工作.
  * 可以**按顺序**运行一系列工作. 

按照上面的Runner需求, 我们设计了一个Runner 类型.

```go
package runner

// 定义一个Runner 开放类型
type Runner struct{
    interrupt chan os.Signal    // 监听系统终端信号
    timeout <-chan time.Time    // 监听超时信号
    complete chan error         // 处理任务完成信号
    tasks []func(int)           // tasks是一组按索引依次执行的函数.
}
```
Runner 包含一个tasks函数切片, 用来管理要执行的任务. Runner 有三个信号:
  * interrupt 信号: 负责监听系统事件.
  * timeout 信号: 负责监听超时信号.
  * complete 信号: 负责监听每个单独task返回值, 成功还是error

根据Runner的内部四个数据类型, 建立一个factory 函数:
```go
func New(d time.Duration) *Runner {
    return &Runner {
        interrupt: make(chan os.Signal, 1),
        complete: make(chan error),
        timeout: time.After(d)
    }
}
```
关于这个工厂函数:
  * 允许调用者设置超时时间, 设定的超时时间会传给函数`time.After`, `time.After`返回值是一个time.Time的通道, 当设定时间到达后, 会往该通道写值.
  * complete 通道被初始为无缓冲通道, 因为一旦一个task完成或者error out, 它就会像main函数(管理者)发送信号, 一旦信号被接受, Runner应该退出.
  * interrupt 被初始化为缓冲区容量为1的通道, 这样可以保证通道至少能接收一个来自os.Signal的值, 确保runtime发送这个事件不会被堵塞. 因为如果对应的goroutine还没准备好.

既然定义了Runner结构, 一个比较好的代码规范是, 我们也需要定义常见的错误类型:
```go
var (
    ErrTimeout = errors.New("执行超时")
    ErrInterrupt = errors.New("收到系统中断信号")
)
```

Runner需要能够添加不同的tasks, 所以给Runner增加一个Add 方法:
```go
func (r *Runner) Add(tasks ...func(int)) {
    r.tasks = append(r.tasks, tasks...)
}
```
这里的Add方法接收一个名为tasks的可变参数, 接收可以是任意数量的只要满足类型是函数(接收一个整型参数, 不反回任何值)传入.

定义完Add之后, 需要一个方法告诉Runner来执行内部的tasks. 

```go
func (r *Runner) run() error{
    for id, task := range r.tasks{
        select {
            // 当中断事件触发时, 
            case <- r.interrupt :
                signal.Stop(r.interrupt)
                return ErrInterrupt
            default: // do nothing just continue
        }
        task(id)
    }
    return nil
}
```

在go里, 用select关键词来检查接收到的信号, 一般而言, select在没有收到信号的时候是堵塞的, 但加了default分支, 就不会堵塞. 这里我们按照tasks 被添加的顺序依次执行, 每执行一个新任务时, 都会检查有没有收到interrupt信号, 如果有收到, 那么就直接返回ErrInerrupt, 没有就继续执行下面一个任务. 此外, 采用signal.Stop() 可以阻止接收之后的所有事件.


上述基本就是Runner所需要的所有方法了, 还需要一个外部启动Runner的方法:
```go
func (r *Runner) Start() error {
    // 我们希望接收所有中断信号
    signal.Notify(r.interrupt, os.Interrupt)

    // 用不同的goroutine执行不同的任务
    go func() {
        r.complete <- r.run()
    }()

    select {
        // 当任务完成时发出的信号
        case err := <- r.complete:
            return err
        // 当任务处理程序超时时发出的信号
        case <- r.timeout:
            return ErrTimeout
    }
}
```

完整的Runner代码:
```go
package main

import (
	"errors"
	"os"
	"os/signal"
	"time"
)

var (
	ErrTimeout   = errors.New("执行超时")
	ErrInterrupt = errors.New("收到系统中断信号")
)

// 定义一个Runner 开放类型
type Runner struct {
	interrupt chan os.Signal   // 监听系统终端信号
	timeout   <-chan time.Time // 监听超时信号
	complete  chan error       // 处理任务完成信号
	tasks     []func(int)      // tasks是一组按索引依次执行的函数.
}

func New(d time.Duration) *Runner {
	return &Runner{
		interrupt: make(chan os.Signal, 1),
		complete:  make(chan error),
		timeout:   time.After(d),
	}
}

func (r *Runner) Add(tasks ...func(int)) {
	r.tasks = append(r.tasks, tasks...)
}

func (r *Runner) run() error {
	for id, task := range r.tasks {
		select {
		// 当中断事件触发时,
		case <-r.interrupt:
			signal.Stop(r.interrupt)
			return ErrInterrupt
		default: // do nothing just continue
		}
		task(id)
	}
	return nil
}

func (r *Runner) Start() error {
	// 我们希望接收所有中断信号
	signal.Notify(r.interrupt, os.Interrupt)

	// 用不同的goroutine执行不同的任务
	go func() {
		r.complete <- r.run()
	}()

	select {
	// 当任务完成时发出的信号
	case err := <-r.complete:
		return err
	// 当任务处理程序超时时发出的信号
	case <-r.timeout:
		return ErrTimeout
	}
}
```

再写一个main函数:
```go
package main

import (
	"log"
	"os"
	"time"

	"github.com/xlk3099/runner"
)

// 定义超时时间
const timeout = 3 * time.Second

func main() {
	log.Println("任务开始...")

	// 用timeout创建一个新的runner instance
	r := runner.New(timeout)

	// 添加三个Task
	r.Add(createTask(), createTask(), createTask())

	// Run the tasks and handle the result.
	if err := r.Start(); err != nil {
		switch err {
		case runner.ErrTimeout:
			log.Println("由于超时, Runner终止.")
			os.Exit(1)
		case runner.ErrInterrupt:
			log.Println("由于操作系统干扰, 程序终止.")
			os.Exit(2)
		}
	}

	log.Println("Runner执行结束.")
}

// createTask 返回一个函数
func createTask() func(int) {
	return func(id int) {
		log.Printf("执行任务 - #%d.", id)
		time.Sleep(time.Duration(id) * time.Second)
	}
}
```

让上述代码自由运行, 我们得到输出:
```
➜  xlk3099 go run main.go
2018/04/15 22:24:21 任务开始...
2018/04/15 22:24:21 执行任务 - #0.
2018/04/15 22:24:21 执行任务 - #1.
2018/04/15 22:24:22 执行任务 - #2.
2018/04/15 22:24:24 由于超时, Runner终止.
exit status 1
```
在运行一段时间, 我们按ctrl+c, 得到输出:
```
➜  xlk3099 go run main.go
2018/04/15 22:20:08 任务开始...
2018/04/15 22:20:08 执行任务 - #0.
2018/04/15 22:20:08 执行任务 - #1.
^C2018/04/15 22:20:09 由于操作系统干扰, 程序终止.
exit status 2
```

当然, 把超时时间设置成>3的话, 我们会看到:
```
➜  xlk3099 go run main.go
2018/04/15 22:22:46 任务开始...
2018/04/15 22:22:46 执行任务 - #0.
2018/04/15 22:22:46 执行任务 - #1.
2018/04/15 22:22:47 执行任务 - #2.
2018/04/15 22:22:49 Runner执行结束.
```

这个只是一个最基本的Runner模型, 还可以有更进一步的扩展, 大家可尝试研究. 