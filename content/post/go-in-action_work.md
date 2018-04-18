---
title: "Go语言实战读书笔记（十六）: 并发模式 - Work"
date: 2018-04-16T18:45:06+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["Go In Action"]
---

work 包利用了无缓冲通道来创建一个goroutine池, (注意, 上一章我们创建了一个资源池, 这里我们创建的是goroutine池) 这些goroutine执行并控制一组工作, 让其并发执行. 这种case下, 使用无缓冲通道的效果比随意指定一个有缓冲区通道好, 因为这种case下不需要一个工作队列, 也不需要一组goroutine 配合执行. 无缓冲通道的一大好处, 就是保证了两个goroutine之间的数据交换. 使用无缓冲通道的方法允许使用者知道什么时候goroutine池正在执行工作, 如果里面所有的goroutine都忙, 无法接受新的工作, 也能通过通道来告诉调用者.

这么看来, 这章其实叫做goroutine pool更好...😁


```go

// 定义一个Pool
type Pool struct {
    work chan Worker 
    wg sync.WaitGroup
}

type Worker interface {
    Task()
}
```
上面我们定义了一个Pool类型, Pool 有一个内部类型work用来接收Worker类型数据,
还有一个WaitGroup参数, 用来等待所有work 被完成.


Pool还需要有其他方法, 比如工厂函数, 给定大小, 建一个新的goroutine池, 添加新的任务.

```go

func New(size int) *Pool {
    p := Pool {
        work : make(chan Worker),
    }

    p.wg.Add(size)

    for i:=0; i<size; i++{
        go func(){
            for w := range p.work{
                w.Task()
            }
            p.wg.Done()
        }()
    }
    return &p
}
```
Pool 需要一个可以添加任务的方法:
```go
func (p *Pool) Run(w Worker){
    p.work <- w
}
```

外部可以关闭Pool
```go
func (p *Pool) Shutdown() {
    close(p.work)
    p.wg.Wait()
}
```

其实整个逻辑很简单, goroutine Pool里有个无缓冲通道, 当Pool 被创建的时候, 会启动给定数量的goroitines, 同时监听那个无缓冲通道. 外部可以往无缓冲通道里写入任务,
由于一个任务只能被一个接收者(goroutine)接收, 所以剩余的goroutines会继续等待任务, 一旦所有的goroutines都处在忙碌状态, 那么外部也不能再往无缓冲通道里写入任务.

完整的Pool代码:
```go
package pool

import "sync"

// 定义一个Pool
type Pool struct {
	work chan Worker
	wg   sync.WaitGroup
}

type Worker interface {
	Task()
}

func New(size int) *Pool {
	p := Pool{
		work: make(chan Worker),
	}

	p.wg.Add(size)

	for i := 0; i < size; i++ {
		go func() {
			for w := range p.work {
				w.Task()
			}
			p.wg.Done()
		}()
	}
	return &p
}

func (p *Pool) Run(w Worker) {
	p.work <- w
}

func (p *Pool) Shutdown() {
	close(p.work)
	p.wg.Wait()
}
```

测试程序(来自go语言实战):
```go
package main

import (
	"log"
	"sync"
	"time"

	"github.com/xlk3099/work"
)

// names提供了一组用来显示的名字
var names = []string{
	"steve",
	"bob",
	"mary",
	"therese",
	"jason",
}

// namePrinter使用特定方式打印名字
type namePrinter struct {
	name string
}

// Task实现Worker接口
func (m *namePrinter) Task() {
	log.Println(m.name)
	time.Sleep(time.Second)
}

// main是所有Go程序的入口
func main() {
	//使用两个goroutine来创建工作池
	p := work.New(2)
	var wg sync.WaitGroup
	wg.Add(100 * len(names))
	for i := 0; i < 100; i++ { // 迭代names切片
		for _, name := range names {
			// 创建一个namePrinter并提供 // 指定的名字
			np := namePrinter{
				name: name,
			}

			go func() {
				p.Run(&np)
				wg.Done()
			}()
		}
	}
	wg.Wait()
	p.Shutdown()
}

```

work这一章跟Pool其实很像, 区别在于资源池内部的资源是其它resources, 比如DB connection. 但work 里面的Pool资源是goroutines, 负责管理多少个goroutines处理一个任务.

---
附上第7章小结:

* 可以使用通道来控制程序的生命周期.
* 可以使用default来防止select语句阻塞.
* 用缓冲通道可以用来管理一组可复用的资源.
* 使用无缓冲通道可以来创建完成工作的goroutine池.
* 任何时候都可以使用同步通道来让两个goroutine交换数据, 在通道操作完成时一定保证对方接收到了数据.