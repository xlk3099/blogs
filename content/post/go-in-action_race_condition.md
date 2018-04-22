---
title: "go实战读书笔记（十二）: race conditions 竞争状态"
date: 2018-04-15T15:00:28+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["Go In Action"]
---

并发中的一大难点就是访问共享资源时, 多个线程或(goroutines in go) 尝试同时对这个资源进行读写, 这种状态叫做 **race condition**, 非常容易引起问题. 一般而言, 对共享资源的操作应该是原子操作, 也就是说, 同一时刻, 最多只能有一个goroutine对共享资源进行读写操作.

</br>

# 竞争状态
---

不啰嗦, 看代码: https://play.golang.org/p/xn7M1ndeINI

> 代码 6-2
```go
// This sample program demonstrates how to create race
// conditions in our programs. We don't want to do this.
package main

import (
	"fmt"
	"runtime"
	"sync"
)

var (
	// counter is a variable incremented by all goroutines.
	counter int

	// wg is used to wait for the program to finish.
	wg sync.WaitGroup
)

// main is the entry point for all Go programs.
func main() {
	runtime.GOMAXPROCS(1)
	// Add a count of two, one for each goroutine.
	wg.Add(2)

	// Create two goroutines.
	go incCounter(1)
	go incCounter(2)

	// Wait for the goroutines to finish.
	wg.Wait()
	fmt.Println("Final Counter:", counter)
}

// incCounter increments the package level counter variable.
func incCounter(id int) {
	// Schedule the call to Done to tell main we are done.
	defer wg.Done()

	for count := 0; count < 2; count++ {
		// Capture the value of Counter.
		value := counter

		// Yield the thread and be placed back in queue.
		runtime.Gosched()

		// Increment our local value of Counter.
		value++

		// Store the value back into Counter.
		counter = value
	}
}

```
输出:
```
Final Counter : 2
```

上述代码对counter进行了4次读写操作, 每个goroutine执行两次, 但是程序停止时, counter的值是2 而不是我们所想的4. 

{{%center%}}![image](https://user-images.githubusercontent.com/1768412/38775864-71a8949e-40bf-11e8-9b47-1ca89b66da82.png)
代码6-2示意图 图片来自 (go实战)
{{%/ center%}}

在代码20行, 我添加了设置最大逻辑处理器为1, 在41行, 加入了runtime包的Gosched, 目的是用于将goroutine从当前线程退出,不然运行速度太快, 我们无法捕获这次race condition. 

go有个工具, 可以在代码里检测竞争状态, 在编译时, 我们加上参数`-race` 即可.
```
go build -race // 用竞争检测器检查程序竞争状态代码

./listing09

==================
WARNING: DATA RACE
Write at 0x0000011d6318 by goroutine 6:
  main.incCounter()
      /Users/qianqian/projects/go_in_action/chapter6/listing09/listing09.go:50 +0x90

Previous read at 0x0000011d6318 by goroutine 7:
  main.incCounter()
      /Users/qianqian/projects/go_in_action/chapter6/listing09/listing09.go:41 +0x6f

Goroutine 6 (running) created at:
  main.main()
      /Users/qianqian/projects/go_in_action/chapter6/listing09/listing09.go:26 +0x75

Goroutine 7 (running) created at:
  main.main()
      /Users/qianqian/projects/go_in_action/chapter6/listing09/listing09.go:27 +0x96
==================
Final Counter: 2
Found 1 data race(s)
```
上述问题一种解决措施就是锁住共享资源, 每次只允许一个goroutine对共享资源进行读写.

</br>

# 锁住共享资源
---

go提供了传统的同步goroutine机制, 就是对共享资源添加锁. go里的atomic包跟sync包提供了很好的解决方案.

1. **atomic 原子函数**
原子函数以底层的加锁机制来同步访问**整形变量**和**整形指针**. 对代码6-2 `incCounter` 利用原子函数做出如下修改可以很轻松的解决竞争问题.

	```go
	// incCounter increments the package level counter variable.
	func incCounter(id int) {
		// Schedule the call to Done to tell main we are done.
		defer wg.Done()

		for count := 0; count < 2; count++ {
			// Safely Add One To Counter.
			atomic.AddInt64(&counter, 1)

			// Yield the thread and be placed back in queue.
			runtime.Gosched()
		}
	}
	```
	这里我们使用了atomic包里的`AddInt64`, 这个函数会同步整形值加法, 方法作用是强制同一时刻只能有一个goroutine运行并完成这个加法操作. 运行新代码后, 我们会得到正确值: 4. 

	另外两个有用原子函数是LoadInt64, 和StoreInt64, 也是提供了安全的读写整形数值的方式. 关于atomic包的更多信息, 可以查看go官网关于atomic的文档:https://golang.org/pkg/sync/atomic

2. **互斥锁**
另一种访问共享资源的方式是试用mutex (mutual exclusion). 互斥锁用于在代码上创建一个临界区, 保证同一时间只有一个goroutine可以执行这段代码, 当然mutex 跟 atomic不一样, 不限于整形数据.

	利用互斥锁修改上述代码:
	```go
	// This sample program demonstrates how to use a mutex
	// to define critical sections of code that need synchronous
	// access.
	package main

	import (
		"fmt"
		"runtime"
		"sync"
	)

	var (
		// counter is a variable incremented by all goroutines.
		counter int

		// wg is used to wait for the program to finish.
		wg sync.WaitGroup

		// mutex is used to define a critical section of code.
		mutex sync.Mutex
	)

	// main is the entry point for all Go programs.
	func main() {
		// Add a count of two, one for each goroutine.
		wg.Add(2)

		// Create two goroutines.
		go incCounter(1)
		go incCounter(2)

		// Wait for the goroutines to finish.
		wg.Wait()
		fmt.Printf("Final Counter: %d\n", counter)
	}

	// incCounter increments the package level Counter variable
	// using the Mutex to synchronize and provide safe access.
	func incCounter(id int) {
		// Schedule the call to Done to tell main we are done.
		defer wg.Done()

		for count := 0; count < 2; count++ {
			// Only allow one goroutine through this
			// critical section at a time.
			mutex.Lock()
			{
				// Capture the value of counter.
				value := counter

				// Yield the thread and be placed back in queue.
				runtime.Gosched()

				// Increment our local value of counter.
				value++

				// Store the value back into counter.
				counter = value
			}
			mutex.Unlock()
			// Release the lock and allow any
			// waiting goroutine through.
		}
	}
	```
	mutex使用方式很简单, 在需要保护的区域, 加上mutex.Lock(), 代码运行完毕后, 再mutex.Unlock()即可. 这里为了使保护区域明显, 在Lock跟Unlock之间加入了Scope`{}`.

	虽然原子函数跟互斥锁都能解决竞争状态, 但事实上, 它们并没有让编写并发程序变得更简单, 更有趣. 下一节我们会介绍channel, 用go的方式来解决共享资源的访问.