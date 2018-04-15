---
title: "Go语言实战读书笔记（七）: struct 结构类型"
date: 2018-04-14T16:41:13+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["Go In Action"]
---

在介绍篇就强调过, go 是静态语言类型. 当编译器了解到数据类型信息的时候, 它可以保障程序用一种安全的方式在处理数据.
数据的类型告诉了编译器两个关键的信息: 1. 要分配多少内存.  2. 这段内所表示的含义.
在go 很多内置类型, 它们的命名上就已经提供了这两点信息, 比如 `int8`, `int32`, `int64`, `float32`, `float64`等等.

# 自定义类型

---

* 声明跟初始化

go 允许用户自己定义数据类型. 最常见的是两种方式定义用户自己的数据类型. 一种是使用关键词`struct`.

```go
// 定义一个新类型 user.
type user struct{
    name    string
    email   string
    mobile  string
    admin   bool
}

// 声明一个用户变量.
var bill user
```

在声明变量的时候, 变量的值总是被初始化的. 如果没有提供自己的数值, 该变量被初始化为其对应的0值. 在go语言里, 任何时候, 定义一个变量并使其初始化为0值, 符合规范的做法是使用 关键词`var` 来定义. 如果该变量需要被初始化其它值, 那我们应该使用短变量声明模式.

```go
lisa := user {
    name: "Lisa",
    email: "lisa@email.com",
    mobile: "123121213",
    admin: false,
}
```
用来初始化的时候, 上述一种常见的声明方式, 对每一个field进行复制的时候, 同时声明了field的名字, 还一种初始化赋值方式, 忽略了所有的field名, 但要求按照定义类型里的filed顺序排序.

```go
lisa := user {"Lisa", "lisa@email.com", 123, false}
```
老实说, 第一种方式更为常见, 阅读性也更佳.

* 类型嵌套

go 语言允许定义一个新的类型, 并且其"field"可以是其它自定义类型.

```go
// 定义一个新的类型, 并以user 作为其field 之一.
type admin {
    person user
    level string
}
```
在这里 admin 有个field person, 其数据类型为刚定义的user.

* 重定义已有类型

另一种方式创建自定义类型是重新定义已有类型.

```go
type Duration int64
```

在这里我们定义一个新的类型`Duration`, 实际它是基于内置类型`int64`.
虽然`Duration`值是`int64`, 但在程序中, 并不能把`Duration`的变量直接复制给`int64`的变量, 反之亦然, 不然编译器还是会报错, 还是需要通过类型转换.

重定义已有类型非常有用, 可以通过重定义已有类型, 来给已有类型加上更多函数, 据我个人经验, 数据转化中是非常有用的, 一大方面便是JSON Marshal 跟 Unmarshal. 在golang 1.9 之后加入了 `alias`, 顾名思义, 别名只是对已有类型的另一种称呼, 虽然很方便, 但他不能增加新的函数, 之后会有单独一章介绍alias.

* 给自定义类型添加方法

方法给自定义类型增添了用户行为, 方法就是函数但在关键词func跟函数名中加了额外的参数. 那个参数被称为接收者(or receiver)

函数:

```go
func min(a, b int) int{
    if a < b {
        return a
    }
    return b
}
```
这里 `min` 就是一个函数, 接受两个input 参数, 并输出其中较小值.

方法:

```go
// 以我们上文定义的用户为例, 给用户增加一个方法: shout 用来喊出他的名字.
type user struct{
    name string
    email string
}

func (u user) shout (){
    fmt.Println(u.name)
}

func main(){
    bob := user{
        name: "bob",
        email: "abc@abc.com",
    }
    bob.Shout()
}
```
调用方法也非常简单, 声明自定义类型变量后, 在后面加上`.`, 就可以调用该类型方法. 跟从`package` 里调用一个函数很类似.

在go中, 方法有两种接收者, 一是值, 二是指针. 

接着上面的例子.

```go
package main

import (
	"fmt"
)

type user struct {
	name  string
	email string
}

func (u user) shout() {
	fmt.Println(u.name)
}

func (u *user) showEmail() {
	fmt.Println(u.email)
}

func main() {
	bob := user{
		name:  "bob",
		email: "abc@abc.com",
	}
	bob.shout()
	bob.showEmail()
}

// output
// bob
// abc@abc.com
```
在这里, 细心的大家可能会发现, 为什么showEmail()的接收者是指针类型, 可bob是值类型, bob也能调用showEmail(). 这是go编译器提供的一个便利,它会自动把接收者类型转换成满足该方法合法的接收者类型.

# 类型的本质

---

在我们给自定义类型添加方法的时候, 是使用值传递还是指针传递? 一个原则是, 假如给这个类型添加或者删除某个值, 是要创建新值, 还是更改当前值? 如果是前者, 那么该类型方法适合值接收者. 如果是后者, 那么适合使用指针接收者. 这同样适用于程式里函数的传递, 保持传递类型的一致性很重要.

1. 原始数据类型
    go 提供了很多内置类型, `int`, `boolean`, `string`等. 这些类型是原始类型, 因此当对这些值进行操作的时候, 一个新的值应该被创建. 基于这个本质, 内置类型在方法或者函数里的传递应该是值传递. 举个例子, string的一个函数Trim.
    ```go
    func Trim(s string, cutset string) string{
        if s == "" || cutset == "" {
            return s
        }
        return TrimFunc(s, makeCutsetFunc(cutset))
    }
    ```
    Trim 函数的作用是, 给定一个字符串, 在给定一个字符集合, 从该字符串中剔除所有在字符集合里的字符并返还. 因为最终目标是生成一个新的字符串, 而且不想让原始字符串受到影响, 所以这里采用的是值传递.

2. 引用类型
    在go里, 引用类型是指slice, map, channel, interface 以及函数这些类型. 当我们声明一个变量是上述这些类型之一的时候, 生成的变量值其实被曾为header value. 所有的head values(来自这些引用类型)都包含一个指向底层数据的指针, 之前我们在切片跟map里已经强调过, 当然也有其他几个不同的成员用来管理底层结构, 比如slice 里面的len, cap等. 所以包含引用类型跟原始类型一致,也是通过值传递的.

3. 结构类型
    结构类型我们上面已经接触到了, 里面包含一系列其它不同的数据类型, 可以是原始的, 也可以是引用类型.
    关于结构类型按值传递还是指针传递, 结构类型按值传递还是指针传递比较复杂. 单单靠方法或者函数会不会修改传入值并不能决定采用值传递还是指针传递, 但有几个基本原则(按重要性排序):
    1. struct本身性质: 如果struct本身就是用来share的, 那不管怎样, 使用指针, 比如`File`, `DB`.
    2. 如果struct size 很大: 用指针, 道理很简单, 因为这时候再使用值传递代价很大.
    3. 如果变量不能被更改,只能新建, 那用值传递, 反之使用指针.

    有人可能会问, 干嘛这么麻烦, 全部使用引用传递不就得了...事实上, 全采用指针传递会引起各种冲突,GC回收不当, 而且go语言里面`pass by value`比`pass by pointers`高效. 所以不要嫌麻烦, case by case 处理.