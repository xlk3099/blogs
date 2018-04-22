---
title: "go实战读书笔记（十五）: 并发模式 - Pool"
date: 2018-04-16T15:49:32+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["Go In Action"]
---

上一节介绍了用go实现Runner, 这一节会介绍利用go的缓冲区通道跟goroutines实现资源池开发. 资源池用来管理任意数量的goroutine之间共享以及独立使用的资源. 这种模式在共享数据库链接或者内存缓冲区非常有用, 如果goroutine需要从池里得到资源中一个, 它可以从资源池里申请, 使用完毕后返还.


先构建一个资源池类型:

```go
// Pool 管理一组可以安全地在多个goroutine之间共享的资源.
// 被管理的资源必须实现io.Closer接口.
type Pool struct {
    m sync.Mutex
    resources chan io.Closer
    factory func() (io.Closer, error)
    closed bool
}
```

* `m` 定义了一个mutex, 保证了在多个goroutine访问资源池时, 池内的值是安全的.
* `resources` 是一个声明为io.Closer的通道, 其为有缓冲通道, 用来保存共享资源. 通道类型是个接口, 所以任意实现了io.Closer接口的数据类型都能作为资源类型.
* `factory` 是一个工厂函数类型, 当池需要新的资源时, 可以通过这个函数实现. 这个函数实现由调用Pool 的使用者提供.
* `closed` 是一个标志, 用来表示Pool 是否被关闭.

老规矩, 定义常见错误:

```go
// 试图访问资源池, 但资源池已经关闭错误.
var ErrPoolClosed = errors.New("资源池已关闭.")
```

从资源池的定义作用, 我们可以分析出资源池大概需要支持下列几种方法:

  1. 一个工厂函数, 新建一个资源池, 调用者可以设定资源池大小, 自己设定资源类型.
  2. 外部goroutine能从资源池里获取一个资源, 如果所有资源都处于繁忙状态, 创建新资源.
  3. 外部goroutine完成任务, 能释放资源回到资源池.
  4. 关闭资源池, 同时关闭所有现有资源防止被goroutine使用.

根据上面定义的要求, 我们给Pool添加方法

```go
func New(fn func() (io.Closer, error), size uint) (*Pool, error) {
    if size <= 0 {
        return nil, errors.New("创建资源池失败, 给定的资源池大小太小.")
    }

    return &Pool{
        factory: fn,
        resources: make(chan io.Closer, size),
    }, nil
}
```

`New` 方法是一个工厂函数, 它接收调用者传入的资源创建函数, 以及所设定的资源池大小. 方法中对size进行检测, 如果size 不大于0, 那么返回nil, 以及报错. 如果一切合规, 建一个新的resources 缓冲通道, 大小为用户指定的大小, 并返回一个新建的资源池指针.

```go
func (p *Pool) Acquire() (io.Closer, error){
    select {
        case r, ok := <- p.resources:
            log.Println("尝试获取:", "共享资源.")
            if !ok {
                return nil, ErrPoolClosed
            }
            return r, nil
        // 因为没有空闲资源, 创建新资源.
        default:
            log.Println("尝试获取:", "新资源")
            return p.factory()
    }
}
```

`Acquire`方法, 对应资源池会检查自己的resource通道, 是不是有空闲资源. 在这里我们额外添加了一个检测, 如果通道关闭, 那么直接返回错误, 资源池已经关闭. 如果资源池处于忙碌状态, 为了不block select语句, 再加上需求之一是资源池忙碌, 则创建新的资源. 所以我们额外添加了default 语句, 调用用户给定的工厂函数, 来创建新的资源.

```go
func (p *Pool) Release (r io.Closer) {
    // 保证本次操作和Close操作的安全
    p.m.Lock()
    defer p.m.Unlock()

    // 如果池已经被关闭, 销毁这个资源
    if p.closed() {
        r.Close()
        return
    }

    select {
        case p.resources <- r:
            log.Println("释放资源到资源池")

        // 如果资源池已满, 则关闭这个资源
        default:
            log.Println("资源池已满, 销毁资源")
            r.Close()

    }
}
```

关于`Release`方法, 这里我们给Release 添加了mutext, 主要是为了保护p.closed()读取, 保证同一时刻, 不会有其它goroutine调用Close方法去写p.close. 这里我们需要检查pool是否是关闭状态, 因为我们不想往一个已经关闭的通道里发送数据, 这样会引起崩溃. 最下面的select语句则是Release方法主要逻辑, 尝试往资源池里释放资源, 资源池已满就直接销毁.

```go
func (p *Pool) Close() {
    // 保证操作与Release操作的安全
    p.m.Lock()
    defer p.m.Unlock()

    // 如果pool 已经被关闭了, 什么也不用做
    if p.closed {
        return
    }

    // 清空通道资源之前, 将通道关闭
    // 不然会发生死锁
    close(p.resources)

    // 关闭资源
    for r:= range p.resources {
        r.Close()
    }
}
```
`Close`则是Pool的结束语句, 逻辑也写在了代码注释里.

完整的Pool 代码:

```go
package pool

import (
    "errors"
    "io"
	"log"
	"sync"
)

// ErrPoolClosed defines the error pool is closed.
// 试图访问资源池, 但资源池已经关闭错误.
var ErrPoolClosed = errors.New("资源池已关闭.")

// Pool 管理一组可以安全地在多个goroutine之间共享的资源.
// 被管理的资源必须实现io.Closer接口.
type Pool struct {
	m         sync.Mutex
	resources chan io.Closer
	factory   func() (io.Closer, error)
	closed    bool
}

// New is a factory function return a new Pool instance.
func New(fn func() (io.Closer, error), size uint) (*Pool, error) {
	if size <= 0 {
		return nil, errors.New("创建资源池失败, 给定的资源池大小太小.")
	}

	return &Pool{
		factory:   fn,
		resources: make(chan io.Closer, size),
	}, nil
}

// Acquire will help caller get a resource from Pool.
func (p *Pool) Acquire() (io.Closer, error) {
	select {
	case r, ok := <-p.resources:
		log.Println("尝试获取:", "共享资源.")
		if !ok {
			return nil, ErrPoolClosed
		}
		return r, nil
	// 因为没有空闲资源, 创建新资源.
	default:
		log.Println("尝试获取:", "新资源")
		return p.factory()
	}
}

// Release will release caller's resource back to Pool.
func (p *Pool) Release(r io.Closer) {
	// 保证本次操作和Close操作的安全
	p.m.Lock()
	defer p.m.Unlock()

	// 如果池已经被关闭, 销毁这个资源
	if p.closed {
		r.Close()
		return
	}

	select {
	case p.resources <- r:
		log.Println("释放资源到资源池")

	// 如果资源池已满, 则关闭这个资源
	default:
		log.Println("资源池已满, 销毁资源")
		r.Close()

	}
}

// Close will close pool.
func (p *Pool) Close() {
	// 保证操作与Release操作的安全
	p.m.Lock()
	defer p.m.Unlock()

	// 如果pool 已经被关闭了, 什么也不用做
	if p.closed {
		return
	}

	// 清空通道资源之前, 将通道关闭
	// 不然会发生死锁
	close(p.resources)

	// 关闭资源
	for r := range p.resources {
		r.Close()
	}
}
```

编写测试程序:

```go
package pool

import (
	"errors"
	"io"
	"log"
	"sync"
)

// ErrPoolClosed defines the error pool is closed.
// 试图访问资源池, 但资源池已经关闭错误.
var ErrPoolClosed = errors.New("资源池已关闭.")

// Pool 管理一组可以安全地在多个goroutine之间共享的资源.
// 被管理的资源必须实现io.Closer接口.
type Pool struct {
	m         sync.Mutex
	resources chan io.Closer
	factory   func() (io.Closer, error)
	closed    bool
}

// New is a factory function return a new Pool instance.
func New(fn func() (io.Closer, error), size uint) (*Pool, error) {
	if size <= 0 {
		return nil, errors.New("创建资源池失败, 给定的资源池大小太小.")
	}

	return &Pool{
		factory:   fn,
		resources: make(chan io.Closer, size),
	}, nil
}

// Acquire will help caller get a resource from Pool.
func (p *Pool) Acquire() (io.Closer, error) {
	select {
	case r, ok := <-p.resources:
		log.Println("从资源池获取共享资源.")
		if !ok {
			return nil, ErrPoolClosed
		}
		return r, nil
	// 因为没有空闲资源, 创建新资源.
	default:
		log.Println("无空闲资源, 创建新资源")
		return p.factory()
	}
}

// Release will release caller's resource back to Pool.
func (p *Pool) Release(r io.Closer) {
	// 保证本次操作和Close操作的安全
	p.m.Lock()
	defer p.m.Unlock()

	// 如果池已经被关闭, 销毁这个资源
	if p.closed {
		r.Close()
		return
	}

	select {
	case p.resources <- r:
		log.Println("释放资源到资源池")

	// 如果资源池已满, 则关闭这个资源
	default:
		log.Println("资源池已满, 销毁资源")
		r.Close()

	}
}

// Close will close pool.
func (p *Pool) Close() {
	// 保证操作与Release操作的安全
	p.m.Lock()
	defer p.m.Unlock()

	// 如果pool 已经被关闭了, 什么也不用做
	if p.closed {
		return
	}

	// 清空通道资源之前, 将通道关闭
	// 不然会发生死锁
	close(p.resources)

	// 关闭资源
	for r := range p.resources {
		r.Close()
	}
}

```

输出:
```
 > go run main.go
2018/04/16 18:19:15 无空闲资源, 创建新资源
2018/04/16 18:19:15 创建新的DB连接: 1
2018/04/16 18:19:15 无空闲资源, 创建新资源
2018/04/16 18:19:15 无空闲资源, 创建新资源
2018/04/16 18:19:15 无空闲资源, 创建新资源
2018/04/16 18:19:15 创建新的DB连接: 3
2018/04/16 18:19:15 创建新的DB连接: 4
2018/04/16 18:19:15 无空闲资源, 创建新资源
2018/04/16 18:19:15 创建新的DB连接: 5
2018/04/16 18:19:15 创建新的DB连接: 2
2018/04/16 18:19:15 查询ID[2] DB连接ID[5]
2018/04/16 18:19:15 释放资源到资源池
2018/04/16 18:19:15 查询ID[0] DB连接ID[2]
2018/04/16 18:19:15 释放资源到资源池
2018/04/16 18:19:15 查询ID[4] DB连接ID[1]
2018/04/16 18:19:15 资源池已满, 销毁资源
2018/04/16 18:19:15 关闭DB连接: 1
2018/04/16 18:19:15 查询ID[3] DB连接ID[4]
2018/04/16 18:19:15 资源池已满, 销毁资源
2018/04/16 18:19:15 关闭DB连接: 4
2018/04/16 18:19:15 查询ID[1] DB连接ID[3]
2018/04/16 18:19:15 资源池已满, 销毁资源
2018/04/16 18:19:15 关闭DB连接: 3
2018/04/16 18:19:15 关闭程序.
2018/04/16 18:19:15 关闭DB连接: 5
2018/04/16 18:19:15 关闭DB连接: 2
```