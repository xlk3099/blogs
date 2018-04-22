---
title: "go实战读书笔记（八）: interface 接口"
date: 2018-04-14T20:31:15+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["Go In Action"]
---

多态是所有现代高级编程语言最基本的一个特性. 多态指代码能根据类型的具体实现采取不同行为的能力, 在go里, 多态的实现通过实现interface(接口)定义的不同函数.

接口是一个类型, 但它是一个抽象类型, 它定义了一些列方法, 跟其他数据类型不一样. 比如int, string 我们知道它们是什么, 也知道它们那些方法的具体实现. 相比而言, 接口只告诉我们这个接口能做什么, 调用它的那些函数需要哪些类型的参数, 返回时什么类型, 但它具体怎么做的, 并不知道, 也没有被具体实现.

接口的一大好处就是把做什么跟具体实现分隔开来, 使得程序更为灵活, 并支持多态.

看个简单的例子.
```go
package main 

import (
    "bytes"
    "fmt"
    "io"
    "os"
)

func main() {
    var b bytes.Buffer
    b.Write([]byte("Hello"))
    fmt.Fprintf(&b, "World!")
    io.Copy(os.Stdout, &b)
}
```
我们看一下`fmt.Fprintf` 这个函数.
```go
func Fprint(w io.Writer, a ...interface{}) (n int, err error) {
    p := newPrinter()
    p.doPrint(a)
    n, err = w.Write(p.buf)
    p.free()
    return
}
```
`fmt.Fprintf` 接受`io.Writer`作为它第一个参数.

我们再看一下`io.Writer`定义:
```go
type Writer interface {
        Write(p []byte) (n int, err error)
}
```

`io.Writer` 是一个接口, 里面有一个Write函数, 也就是说, 任何类型定义了Write这样的方法, 我们就说该类型实现了io.Writer接口 `=>` 该类型就能作为fmt.Fprintf() 函数的第一个参数. 这里因为bytes.Buffer实现了io.Writer的接口, 所以能作为fmt.Fprintf()的参数.

</br>

# 接口内部实现
---

我们上文讲到, 一旦一个类型实现了指针定义的函数, 我们就称该类型实现了指针的接口. 我们这里称该类型为具现类型, 这些函数到具现类型也对应的成为这些具现类型的方法.

接口内部有两个数据, 分别是两个指针, 一个指向具现化类型的iTable, 另一个指向具现化类型的值.

我们在上一章讲到过, 一个方法的接收者可以是指针, 或者是值. 在接口这里我们得分开研究.

{{%center%}}
![image](https://user-images.githubusercontent.com/1768412/38768422-30186b74-4026-11e8-8a42-077f7401b689.png)
__当值类型实现了一个接口所有方法示意图__
{{%/ center %}}
我们注意到, 当notifier被具现为user的时候, notifier的第一个指针, 指向了值类型(user)的iTable. 第二个指针则指向了user 所储存值.

{{%center%}}
![image]![image](https://user-images.githubusercontent.com/1768412/38768620-501c3d8a-4029-11e8-9ade-91efe2daa454.png)
__当指针类型实现了一个接口所有方法示意图__
{{%/ center %}}
这图, 我们发现, 第一个指针指向了`指针类型`(*user)的iTable, 而第二个指针指向不变.

接着上面两张图, 我们看一下下面一段代码

```go
package main

import (
    "fmt"
)
// 定义一个接口, notifier
type notifier interface {
    notify()
}

// 定义一个具现化类型 user
type user struct {
    name  string
    email string
}

// notify 实现了notifier的 interface, 但接收者为user 指针.
func (u *user) notify() {
    fmt.Printf("Sending user email to %s<%s>\n",
        u.name,
        u.email)
}

func main() {
    // 建造一个user实例.
    u := user{"Bill", "bill@email.com"}

    sendNotification(u)
    // 编译器返回报错.
    // ./listing36.go:32: cannot use u (type user) as type
    //                     notifier in argument to sendNotification:
    //   user does not implement notifier
    //                          (notify method has pointer receiver)
}

// sendNotification 接受一个实现了notifier interface的类型作为参数.
func sendNotification(n notifier) {
    n.notify()
}
```

运行这段代码, 编译器会报错, 说user没有实现notifier这个借口, 机智的小伙伴们, 结合上面两张图应该已经发现问题了. 但要具体理解为什么, 我们需要明白方法集(method set) 这个概念.

</br>

# 方法集
---

go规范里定义的方法集规则.

| Values        | Methods Receivers           |
| :------------:|:---------------------------:|
| T             | (t T)                       |
| *T            | (t T) and (t *T)            |

也就是说, T类型的值的方法集只包含值接收者声明的方法, 而指向T类型的指针的方法集合既包括值类型的接收者声明的方法, 也包含指正接收者声明的方法. 咋一看比较容易混淆. 让我们从接收者角度分析这个规则.

| Methods Receivers           | Values        |
|:---------------------------:| :------------:|
| (t T)                       | T and T*      |
| (t *T)                      | *T            |
换个角度从接收者角度思考, 也就是说, 如果使用指针接收者来实现一个接口, 那么只有指向那个类型的指针才能够实现对应接口. 如果使用值接收者来实现一个接口, 那么那个类型的指针和值都能实现对应的接口.

总结: **如果值做为方法接收者来实现接口函数, 那么该类型的值和指针都表示实现了该接口. 但如果指针作为方法接收者来实现接口函数, 那么只有该类型的指针才实现该接口.**
