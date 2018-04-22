---
title: "go实战读书笔记（九）: Type embedding 类型嵌入"
date: 2018-04-14T21:51:35+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["Go In Action"]
---

go开发一段时间, 大家就会发现, 如果我想改已有原始类型添加方法怎么办? 可以通过类型嵌入来实现.

类型嵌入是将已有类型直接声明在新的结构类型里, 而不给它起任何的名字.
通过类型嵌入, 与内部类型相关的标识符会提升到外部类型上.

示例:

```go
package main

import (
    "fmt"
)

// 定义一个用户类型
type user struct {
    name string
    email string
}

// noitfy 实现了一个可以通过user类型的指针调用的方法
func (u *user) notify() {
    fmt.Printf("Sending user email to %s<%s>\n", u.name, u.email)
}

type admin struct {
    user // 嵌入类型
    level string
}

func main() {
    // 创建一个admin实例
    ad := admin {
        user: user{
            name: "john smith",
            email: "john@yahoo.com",
        },
        level: "super",
    }

    // 可以直接访问内部类型方法
    ad.user.notify()

    // 或者通过外部类型访问
    ad.notify()
}
```
> ad.user.notify() 这行代码, 对于外部类型而言, 内部类型总是存在的, 虽然没有指定内部类型的字段名, 但是可以直接使用内部类型的类型名来访问内部类型的值.

类型嵌入还有一个强大的地方, 当内部类型实现某个接口, 那么, 外部类型也实现了该接口.

```go
func sendNotification(n notifier){
    n.notify()
}

func main() {
    // 创建一个admin实例
    ad := admin {
        user: user{
            name: "john smith",
            email: "john@yahoo.com",
        },
        level: "super",
    }

    sendNotification(&ad)
}

// output
Sending user email to john smith <john@yahoo.com>
```

问题来了, 万一外部类型声明了自己的notify方法会如何?
```go
type admin struct{
    user
    level string
}

func (a *admin) notify(){
    fmt.Printf("Sending admin email to %s<%s>\n", a.name, a.email)
}


func main() {
    // 创建一个admin实例
    ad := admin {
        user: user{
            name: "john smith",
            email: "john@yahoo.com",
        },
        level: "super",
    }

    sendNotification(&ad)

    ad.user.notify()

    ad.notify()
}
```

机智的小伙伴可能已经猜到了结果...

```
Sending admin email to john smith<john@yahoo.com>
Sending user email to john smith<john@yahoo.com>
Sending admin email to john smith<john@yahoo.com>
```
这就表明, 如果外部类型实现了跟内部类型同样的方法, 那么外部类型的优先级>内部类型的优先级. 不过由于内部类型是一直存在的, 因此还是可以通过访问内部类型来调用内部类型的方法.

类型嵌入其实运用真的很广, 比如写个DB Wrapper, 给该DB扩展一些方法, 或者某些内置的类型的Wrapper扩展一些方法都是很常见的.