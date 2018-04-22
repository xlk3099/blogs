---
title: "go实战读书笔记（一） ：go简介 "
date: 2018-04-02T22:08:46+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["go In Action"]
---

# 开发速度
---
相比C跟C++，go的编译器更为简洁快速，在编译一个go项目的时候，编译器只需要查找直接引用的包，而不需要像java，C或者C++那样一层层引入全部的依赖包。很多go项目的编译速度都是小于一秒，在现代机器上，go自身的编译时间也是不到20秒。。。像ETH上一定规模的项目，编译也是10秒不到。

go是静态语言。跟javascript或者python这种动态语言不同。静态类型语言的一大好处就是，提供了类型安全保障（type safety）。动态语言数据类型直到运行时才会被决定，而静态语言在编译时编译器就会检查，如果有错则会提前告知.

</br>

# 并发
---
编程里面一大难点便是最大程度利用系统资源。现代电脑有很多内核跟线程，但大多数编程语言并没有提供便捷的方式来有效地利用这些系统资源，它们通常要求程序员写很多的线程间的同步代码，非常容易造成错误。

go最强特色就是它对于并发的支持性。go里引入了goroutine这种概念，，可以看成轻量级的线程、goroutines跟线程相比占用的内存跟所需的代码小很多。go使用channels在goroutines之间进行通信传递数据, channels产生了一种新的编程机制，goroutines彼此之间传递数据，而不是相互竞争使用同样的数据。

goroutines 的一大特色是，它会与其他的goroutine同时运行，包括main函数也是一个goroutine。在其他语言中，通常是使用多个线程实现并发来完成一个任务，但在go中，一个线程可以同时运行很多个goroutines。每一个goroutine 占用的内存非常小，大概4～5kb左右，也就是说在一个4GB内存电脑上，理论上可以运行到上百万个goroutines。

channels是在不同goroutine实现通信到数据类型。大多数编程语言到并发中，一大难点就是处理共同到资源，如内存，数据类型等。channels保障了同时只能一个goroutine修改一个数据。

值得注意等是，channels并不提供数据访问保护。如果数据备份（pass by value)通过channels交互，那么每一个goroutine都有自己的数据备份，无论对该数据做出怎样对数据修改，对于其它goroutine而言，都是安全的。但如果是指针通过channels交互的话，每一个goroutine还是需要同步如果读写会被其它不同的goroutine同时进行。说白了这种case就是需要读写保护。

</br>

# go的类型系统
---
go提供了灵活的，无继承的类型系统。无需降低性能就能最大程度重复使用代码。
go另一大特色是接口，跟java不同的是，go是对行为进行建模，而不是对类型进行建模。在go中，不需要声明某个类型实现哪个接口，只要那个类型实现了该接口对所有行为，就代表该类型实现了该接口。
举个🌰

```go

type A interface {
    MethodA()
}

type C interface{
    MethodC()
}

type B struct{}

func(b *B) MethodA(){
    fmt.Println("Hello, I am actually B")
}

func (b *B) MethodC(){
    fmt.Println("Hello, I am actually B")
}
```
这样B就实现了A接口跟C接口，当我们新建一个A类型或者C时，就可以用类型B来构建

```go
func NewA() A {
    return &B{}
}

func NewC() C{
    return &B{}
}

func main(){
    var a = NewA()
    var c = NewC()
    a.MethodA()
    c.MethodC()
}
// output:
//  Hello, I am actually B
//  Hello, I am actually B
```
完整示例代码: [go playground](https://play.golang.org/p/ZG12dM52qlM)

</br>

# 内存管理
---
在C或者C++中，令开发者一大头疼的就是GC处理, 不适当的内存管理会造成程序的崩溃或者内存泄漏。go有现代的GC回收机制来自动帮你处理这部分复杂工作。