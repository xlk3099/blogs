---
title: "Go语言实战读书笔记（十一）: goroutine"
date: 2018-04-15T12:02:23+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["Go In Action"]
---


之前我们提到过, go 语法和运行时直接内置了对并发的支持(通过goroutine跟channel). go 语言的并发同步模型是来自于一个叫做通信顺讯进程(Communicating Sequential Processes, CSP)的范式. CSP 是一种消息传递模型, 用于在goroutines之间传递消息, 而不是通过对数据加锁来实现同步访问. channel 是用来起到传递的关键数据类型.

# 线程, 进程, 调度器, 逻辑处理器 一些基本概念
---

很多人都解释不清goroutine跟thread的具体区别, 这里还是提一下一些进程, 线程, 调度器等基本概念 以便帮助理解go是如何运用goroutine是如何实现并发的.

* 进程 process: 当一个应用程序启动的时候, OS会为这个应用系统启动一个进程. 可以将进程看做一个包含了应用程序在运行中需要用的的和维护的各种资源容器. 包括内存空间, 文件设备以及线程等等.
* 线程 thread: 一个线程是一个执行空间, 通常一个进程会包含一到多个线程. 这个空间会被OS调度来运行应用程序函数中所写的代码.

    {{%center%}}![image](https://user-images.githubusercontent.com/1768412/38775105-5d96f5bc-40ac-11e8-9930-b1e8b90b0e34.png)应用程序的进程跟线程简单示意图 图片来自(go语言实战)
    {{%/ center%}}

    OS会在物理处理器(大家熟悉的CPU)上调度线程来运行, 不同的是, 在go语言运行时, go会在**逻辑处理器** 上调度goroutine来运行.

* 调度器,逻辑处理器:  go语言运行时, 有自己的调度器(一个复杂的软件), 当goroutine被创建时时, go会将其视为一个独立的工作单元, 并且会被调度到可用的逻辑处理器上执行. 调度器会管理所有的goroutine并且为他们分配执行时间. go的这个调度器在OS之上, 将操作系统的线程与逻辑处理器绑定, 并安排在逻辑处理器上运行goroutine. 调度器在任何时间, 都会全面控制哪个goroutine在哪个逻辑处理上运行. **每个逻辑处理器都只会被绑定到单个操作系统线程**, 在go 1.5 之前, go只给每个应用程序分配一个逻辑处理器, 在go 1.5 之后 go会给每个可用的物理处理器分配一个逻辑处理器.

    {{%center%}}![image](https://user-images.githubusercontent.com/1768412/38775166-292a22e8-40ae-11e8-81e5-0d1c7e93c4cf.png)Go调度器管理goroutine示意图 图片来自(go语言实战)
    {{%/ center%}}
    
    上图演示了线程, 逻辑处理器, 和本地运行队列的关系. 如果创建了一个goroutine, 这个goroutine就会被加到**全局运行队列**之中. 之后, 调度器会将这些goroutine分配给一个处理器, 并放到这个逻辑处理器对应的**本地队列**中. 然后本地队列中的goroutine会一直等待知道自己被分配的逻辑处理器执行.

    第二张图演示了阻塞的情况, 比如打开一个文件. 当一个goroutine做类似操作阻塞系统调用该线程时候, 线程和该goroutine会从逻辑处理器上分离, 该线程会继续阻塞, 等待系统调用返回. 同时, 逻辑处理失去原先绑定的线程, 这时候go调度器会给其配一个新的线程, 然后调度器会从本地队列选取另一个goroutine运行. 一旦之前被阻塞的线程执行完成返回, 对应的goroutine会被放回本地队列, 之前线程也会保存好等待被使用. 

**并发跟并行的区别(concurrency vs parallelism)**:
go 里面谈的并发为主而不是并行, 并行是让不同的代码片段在不同的处理器上同时执行. 并行强调的是同时做很多事情, 并发是同时管理很多事情. 很多时候, 并发效果比并行号, 因为系统资源有限. go 语言设计哲学就是 "less is more".

当然go是支持并行的, 让go程序并行, 必须使用多个逻辑处理器. 当有多个逻辑处理器时, 调度器会将其平均分配到每个逻辑处理器上, 这样就会使goroutine在不同线程上运行. 当然前提是机器得有多个无力处理器.

{{%center%}}
![image](https://user-images.githubusercontent.com/1768412/38775227-e34b34d6-40af-11e8-92ba-c148be892dd4.png) 并发与并行的区别 图片来自(go语言实战)
{{%/ center%}}

# 使用goroutine
---

在go里面, 通过关键词`go` 后面跟已定义函数, 方法, 或者匿名函数来创建一个新的goroutine.
看一个goroutine 简单例子, 代码链接: https://play.golang.org/p/TN0BTVd6yzG

> 代码 6-1
```go
package main

import (
    "fmt"
    "runtime"
    "sync"
)

func main() {
    // 分配最多一个逻辑处理器给调度器使用
    runtime.GOMAXPROCS(1)

    // wg 用来等待程序完成
    // 计数加2, 表示要等待两个goroutine完成
    var wg sync.WaitGroup
    wg.Add(2)

    fmt.Println("Start goroutines.")

    // 创建一个新的goroutine, 运行一个匿名函数
    go func() {
        // 函数退出时, 通知main函数工作完成
        defer wg.Done()

        for count := 0; count < 3; count ++ {
            for char := 'a'; char < 'a' + 26; char ++ {
                fmt.Printf("%c ", char)
            }
        }
    }()

    // 创建一个新的goroutine, 运行一个匿名函数
    go func() {
        // 函数退出时, 通知main函数工作完成
        defer wg.Done()

        for count := 0; count < 3; count ++ {
            for char := 'A'; char < 'A' + 26; char ++ {
                fmt.Printf("%c ", char)
            }
        }
    }()

    fmt.Println("Waiting to Finish")
    wg.Wait()

    fmt.Println("\nTerminating Program")
}
```
输出:
```
Start goroutines.
Waiting to Finish
A B C D E F G H I J K L M N O P Q R S T U V W X Y Z A B C D E F G H I J K L M N O P Q R S T U V W X Y Z A B C D E F G H I J K L M N O P Q R S T U V W X Y Z a b c d e f g h i j k l m n o p q r s t u v w x y z a b c d e f g h i j k l m n o p q r s t u v w x y z a b c d e f g h i j k l m n o p q r s t u v w x y z 
Terminating Program
```

从输出我们可以看到, 第二个被声明的goroutine被调度器安排第一个执行, 所以要引起注意的是, **goroutine的声明顺序并不代表它们的执行顺序**, 而且由于完成该任务所需时间太短, 以致调度器切换到下一个goroutine时, 第一个goroutine就已经就完成了所有任务.
而且我们看到, "Waiting to Finish" 优先于两个goroutine的输出, 这表示, 当创建完goroutine之后, main函数中的代码会继续执行, 这里引发了一个很重要的概念, 在go里面, main函数也是一个goroutine, 不过, 由于main函数是主进程函数, 一旦main函数返还, 那么它会terminate对应的进程, 我们在上文中刚介绍过应用程序进程概念, 所以其他goroutine正在执行或者还没执行就会被打断, 因为整个进程的占用的resource会被release. 因此, 在这里我们加入了WatiGroup, 告诉main函数, 等待两个goroutine完成它们的工作.

**WaitGroup** 是一个计数信号量, 用来记录并维护运行的goroutine. 如果WaitGroup的值大于0, 那么函数Wait就会block. 为了减小WaitGroup的计数, 每个goroutine在退出的时候需要调用Done方法.

关于调度器, 这里有个概念得提出来, 一个正在运行的goroutine在其工作结束前是可以被停止,被之后再被调度的. 调度器这么做是为了防止某个goroutine长时间占用逻辑处理器. 当goroutine占用时间过长, 调度器会停止当前正在运行的goroutine, 并给其他可运行的goroutine机会. 如下图, 第一步, 调度器运行GA(goroutine A), GB 在本地队列里等待. 之后, 调度器交换了GA和GB, 开始运行GB, 而由于GA没有完成, 因此GA重新被放回本地队列. 在第三部, GB完成, 系统销毁GB, 调度器继续运行GA.

{{%center%}}
![image](https://user-images.githubusercontent.com/1768412/38775707-b150f63a-40bb-11e8-8f93-9840ab6b449f.png)
goroutine在逻辑处理器线程上交换示意图 图片来自(go语言实战)
{{%/ center%}}

在代码6-1 底11行中, 因为我们设定了逻辑处理器数量为1, 所以整段代码运行的时候没有并行.
把该行注释掉, 或者设置成更大, 那就使得该程序并行运行, (还是那句话, 前提你电脑有多核), 你会看到两个goroutine几乎同时开始运行.
__注意__ : goplayground 虚拟环境是单核的, 因此你设多大也看不到并行效果.
```
Create Goroutines
Waiting To Finish 
ABCaDEbFcGdHeIfJgKhLiMjNkOlPmQnRoSpT qUrVsWtXuYvZwAxByCzDaEbFcGdHeIfJgKhL iMjNkOlPmQnRoSpTqUrVsWtXuYvZwAxByCzD aEbFcGdHeIfJgKhLiMjNkOlPmQnRoSpTqUrV sWtXuYvZwxyz
Terminating Program
```

本文小结:

* goroutine背后的线程, 进程, 逻辑处理器, 调度器等概念.
* 并发与并行的区别.
* 了结了如何创建goroutine.
* 如何用waitgroup阻塞程序, 直到指定的goroutine都结束工作.
* 如何设置并行.