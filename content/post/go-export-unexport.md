---
title: "Go语言实战读书笔记 (十）: export & unexport"
date: 2018-04-14T22:23:29+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["Go In Action"]
---

go语言在包level支持标识符的公开或者隐藏.

关于如何公开还是不公开的规则很简单, 如果要公开包里的类型, 或者方法, 参数, 只需要将该类型, 方法, 参数的首字母大写. 也就是说, 当包里的标识符首字母大写时, 那么该标识符就被包外代码课件, 如果是小写, 那就不可见.

基本可以概括成下面 几种case.

```go
package model

// userA 未公开, 因为首字母小写,
type userA struct {
    Email string    // 未公开, 因为外部类型userA未公开
    name  string    // 未公开
}

// 未公开, 因为接收者 *ua 未公开
func(ua *userA) Email() string{
    return ua.Email
}

// UserB 公开
type UserB struct{
    Email string    // 公开
    name string     // 未公开
}

// Email 方法公开
func (ub *userB) Email() string{
    return ub.Email
}

// updateEmail 未公开
func (ub *userB) updateEmail (newEmail string) {
    ub.Email = newEmail
}
// c 未公开, 因为c小写
var c UserB

// D 公开, D大写, 外部能访问D.Email, 但不能访问D.name
var D UserB

// E 公开, 外部能访问E.Email 但不能访问E.name
var E userA
```

基本就这些把, 但一个比较好的go practice是, 任何被公开的标识符, 都应该给其加上documentation说明其作用.

# Chapter 5 小结

---

* 使用关键字struct 或者通过重命名已存在的类型, 可以自定义用户类型.
* 方法提供了一种给用户定义的数据类型增加行为的方式.
* 设计类型是需要确认类型的本质是原始的还是非原始的(可改动).
* 接口是声明了一组行为并支持多态的抽象类型.
* 类型嵌入提供了扩展已有类型的能力, 而无需使用继承.
* 标识符要么是从包里公开的, 要么是未公开的, 通过首字母大小写来达到公开未公开效果.