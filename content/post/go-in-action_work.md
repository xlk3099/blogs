---
title: "Go语言实战读书笔记（十五）: 并发模式 - Work"
date: 2018-04-16T18:45:06+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["Go In Action"]
---

work 包利用了无缓冲通道来创建一个goroutine池, (注意, 上一章我们创建了一个资源池, 这里我们创建的是goroutine池) 这些goroutine执行并控制一组工作, 让其并发执行. 这种case下, 使用无缓冲通道的效果比随意指定一个有缓冲区通道好, 因为这种case下不需要一个工作队列, 也不需要一组goroutine 配合执行. 无缓冲通道的一大好处, 就是保证了两个goroutine之间的数据交换. 使用无缓冲通道的方法允许使用者知道什么时候goroutine池正在执行工作, 如果里面所有的goroutine都忙, 无法接受新的工作, 也能通过通道来告诉调用者.

这么看来, 这章其实叫做goroutine pool更好...😁

再用Pool的名字有点俗了, 既然已经讲述了大概的原理, 我们就换个类型吧. 起一段时间刚看了全职猎人动漫, 上面是Pool跟Worker的关系, 我们就使用Guild跟Hunter的关系吧.

```go

// 定义猎人公会
type Guild struct {
    taskboard chan Task
    wg sync.WaitGroup
    members []Member
}

type Task int64
```
上面我们定义了一个猎人公会类型, 猎人公会有一个任务面板(taskboard), 可以接受Task类型数据,
还有一个WaitGroup参数, 用来等待所有Task完成. 这里简单起见, 我们定义Task为int64的重命名类型, 用来表示Task ID.

有了公会我们肯定需要猎人:

```go
type Member interface {
    Exec(Task)
}

type Hunter struct {
    id int
}

func (h *Hunter) Exec(t Task) {
    fmt.Println("猎人执行任务:" t)
}


```
定义一个猎人接口, 接口有个函数Exec(Task), 表示猎人需要执行Task

公会还需要有其他方法, 比如工厂函数, 给定大小, 建造一个新的公会, 指定公会人数, 添加新的任务等等.

```go
// 这里size虽然表示的公会人数大小, 其实是做多能同时执行的goroutines.
func New (size int) * Guild {
    g := Guild {
        taskboard: make(chan Task),
        members : make([]Hunter, size),
    }
    g.wg.Add(size)

    for member := members {
        go func(m Member) {
            for t := range g.taskboard {
                m.Exec(i)
            }
        }(member)
    }
    return &g
}
```
