---
title: "go实战读书笔记（二）：package 包管理"
date: 2018-04-03T22:20:10+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["Go In Action"]
---
# 包的结构跟命名
---
所有go程序都由若干个包组成, 每个包里面可以包含若干go文件.
举个🌰, go自带的http包:
```
    net/http/
        cgi/
        cookiejar/
            testdata/
        fcgi/
        httptest/
        httputil/
        pprof/
        testdata/
```
在这里`http`是一个包,http下面的`cgi`,`fcgi`等等也都是一个个单独的包. 

**_注意_**: 在所有的go文件里, 第一行代码必须是声明该文件所属的包.

## 包命名规范
1. 一般而言, 包的名字都跟自己所在的目录名(文件夹名)一致((测试文件除外), 这也使得查询代码更为方便. 
2. 在go里, 包的名字并不需要独一无二, 因为在导入的时候,采取的方式是全路径.
3. go的包名一般都是小写,由单个单词组成, 没有必要使用`下划线`,或者大小写方式.

除开包的命名规范, 包里的开放函数或者数据类型命名时也要引起一定的注意, 避免跟包名冗余. 在引入包时, 调用者会使用包名来调用包里的内容, 所以一个包里的函数, 或者属性命名也需要适当调整. 比如, golang常见的`bufio`包,里面的`Reader`数据类型, 而不是叫`BufReader`. 调用的时候,`bufio.BufReader` 存在冗余, `bufio`跟`Buf`实际上是一样的意思,显的变扭. 上下代码可能更明显:

    ```go
    package api

    import "fmt"

    func APIGet(){
        fmt.Println("Hello, I am Get function in package api")
    }

    ```

    在这里我们定义了一个包为`api`, 里面有一个函数 `APIGet`, 当另一个go文件导入`api`包,调用`APIGet`时, 会被这样调用:

    ```go
    package main

    import "api"

    func main() {
        api.APIGet()
    }
    ```
    `api.APIGet()`此时就显得不简洁. 
    一个更常见的case就是构造函数. 举个🌰,现在还是有很多人会这样定义数据库副本的构造函数:`NewDB()`, 假设数据库操纵相关的包名是`db`, 那么我们创建一个新的数据库副本的时候, 是这样调用的:
    ```
    db.NewDB()
    ```
    这也不是很简洁. 事实上`db.New()` 就足以.

## main 包
在go的程序里, `main`包时比较独特的存在. go的编译器会试图把main包编译成可执行文件, 所以每一个go程序都有唯一的一个`main`包, 而且这个`main`包里也必须要有一个`main`函数.

# 包的导入
---
在golang里 试用关键词`import`来导入一个包, 包名用双引号修饰.
要注意的是,当导入多个包时, 习惯是讲`import包装在一个导入块中. 举个🌰:
```go
import (
    "fmt"
    "strings"
)
```
golang的编译器试用go的环境变量`GOPATH`来查找磁盘上的包. 标准库的包则是在安装go的位置找到(GOROOT).
`查找顺序`:编译器会首先查找Go的安装目录, 然后才会按顺序呢查找GOPATH变量列出的目录. 如果GOPATH也找不到,那编译器会报错.
反之,如果go文件里面导入了一个不在代码里使用的包时,编译会也会失败报错.


_Tips_: 可以再命令行输入 `go env`  查看关于go的环境变量.

## 远程导入
目前编程环境的大趋势是使用分布式版本控制系统(Distributed Version Control Systems, **DVCS**)来分享跟管理代码. 如github, go工具本身支持从网站获取源代码. e.g.

```go
import "github.com/spf13/viper"
```
go build 编译器会使用GOPATH环境变量搜索这个包,上述对应的包实际磁盘地址 `$GOPATH/src/github.com/spf13/viper`

## 命名导入
当不同目录下的包具有相同名字的时候,会有重名冲突. 举个🌰: 当我们需要`network/convert` 包来转换从网络读取的数据,又需要`file/convert`, 就会同时导入两个名叫`convert`的包. 我们可以通过命名导入来解决冲突.
```go
package main

import(
    "fmt"
    myfmt "mylib/fmt"
)

func main() {
    fmt.Println("Standard Library")
    myfmt.Println("mylib/fmt")
}
```

## `.` 导入
go里面有种特殊的导入, 只在一些特殊情况或者测试时使用. 在导入包时, 在需要导入的包前加上前缀`.`, 这样就可以忽略包名, 直接调用该包所有开放的内容. 举个🌰:

```go
package main

import(
    . "fmt"
)

func main() {
    Println("Standard Library")
}
```

## `_` 导入
go每个包文件都`若干个` `init`函数(注意是n个), 当你导入该go文件时, 假设你只是想调用它的`init`函数, 对其它包里的函数不感兴趣, 这个时候就可以在该包前面加上`_`前缀. 这样可避免由于不使用该包函数而导致的编译错误.
具体例子可以看下面的init函数.

# 包的init函数
---
`init`函数会在程序执行开始的时候被调用. 所有的`init`函数都会在`main`函数之前执行.`init`函数用在初始化变量,引导工作或者是设计`adapter pattern`的时候很有用.

举个数据库驱动的例子. go提供了`database/sql`包, 包里抽象了`sql`的数据库以及操作的基本方法, 基本适用于所有的sql数据库, 比如`mysql`跟`postgresql`. 但使用这个包时, 需要提供具体sql数据库的driver, 比如你实际使用的是postgresql数据库, 那么就需要提供对应的postgresql的driver. 但是我们在实际操作的时候, 只需要使用`database/sql`提供的函数, 而对于driver是怎么实现的, 根本不care. 这时候, postgresql driver包可以在它init函数里调用sql的Register函数, 将其注册到`database/sql`作为实际的数据库类型. 

```go
//postgresql driver
import (
    "database/sql"
)

func init(){
    sql.Register("postgres", new(PostgresDriver))
}
```

```go
package main

import (
    "database/sql"
    _ "github.com/goinaction/code/chapter3/dbdriver/postgres" // 用"_"避免未使用包错误
)

func main(){
    sql.Open("postgres", "mydb")
}

```

