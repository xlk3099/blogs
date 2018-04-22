---
title: "go实战读书笔记（二十一）: benchmark"
date: 2018-04-22T20:54:32+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["Go In Action"]
---

基准测试, 又名性能测试, 很多时候同一问题有多个解决方案, 我们需要查看哪种解决方案性能更好的时候, 基准测试就很有用. 基准测试也可以用来测试某段代码CPU使用效率. 尤其是在`loop` 多的时候, 基准测试就显得尤其有用.

首先要强调的是, 基准测试也是go测试的一种, 也是通过`go test`运行, 跟单元测试一样, 基准测试的文件也是以`_test.go`结尾, 比如我们有个文件叫`client.go`, 那么对应的基准测试文件是`client_test.go`, 基准测试一般也是跟单测放在同一个文件里. 

在运行go test时, 如果我们只跑基准测试, 我们需要加入指令`bench regexp`来声明我们要运行对满足regex pattern的函数基准测试.
如果要只运行基准测试函数, 那么需要加上`-run="none"`指令.

举个基准测试的例子, go中, 将整数值转换成字符串有三种方法.

> fmt.Sprintf("%d", num)
> strconv.FormatInt(num, 10)
> strconv.Itoa(num)

我们可以通过基准测试来了解哪种方法运行更为迅速.

```go
func BenchmarkSprintf(b *testing.B) {
	number := 10

	b.ResetTimer()

	for i := 0; i < b.N; i++ {
		fmt.Sprintf("%d", number)
	}
}

func BenchmarkFormat(b *testing.B) {
	number := int64(10)

	b.ResetTimer()

	for i := 0; i < b.N; i++ {
		strconv.FormatInt(number, 10)
	}
}

func BenchmarkItoa(b *testing.B) {
	number := 10

	b.ResetTimer()

	for i := 0; i < b.N; i++ {
		strconv.Itoa(number)
	}
}
```

运行上面三个基准测试之前, 我们先看一下一个基准测试函数的基本构成:

* 每个基准测试函数都必须以`Benchmark`开头.
* 每个基准测试函数都必须接受一个`testing.B`的指针参数.
* 每个基准测试函数返回nothing...

在看下每个基准函数测试的内部都有一个循环迭代, 其上限是B.N, 循环里面是要测试的函数对象. 之所以采用循环是为了让基准测试框架能准确测试性能, 在一段时间内反复运行被测试函数. 关于`ResetTimer`, 它是为了重置基准测试已经过的时间, 以及重置内存分配的计数器.

运行上面三个基准函数测试:
```
> go test -v -run="none" -bench="." -benchmem
goos: darwin
goarch: amd64
pkg: github.com/goinaction/code/chapter9/listing28
BenchmarkSprintf-4      20000000                86.8 ns/op            16 B/op          2 allocs/op
BenchmarkFormat-4       500000000                3.15 ns/op            0 B/op          0 allocs/op
BenchmarkItoa-4         300000000                4.54 ns/op            0 B/op          0 allocs/op
PASS
ok      github.com/goinaction/code/chapter9/listing28   5.560s
```

上述输出解释:

* `goos`: 表示当前测试运行系统
* `goarch`:` 表示系统架构
* `pkg: github.com/goinaction/code/chapter9/listing28` : 表示测试路径
*  `BenchmarkSprintf-4`, `BenchmarkFormat-4`, `BenchmarkItoa-4` 表示基准测试名, 以及可同时运行的测试数量(默认为4)

我们看到, 在相同的时间内(testing.B的默认值):

* Sprintf 完成了2000万次操作, 每执一次Sprintf转换数字到字符串, 需要86.8ns
* Format 完成了5亿次操作, 每执行一次FormatInt转换数字到字符串, 需要3.15ns
* Iota完成了3亿次操作, 每执行亿次Iota转换数字到字符串, 需要4.54ns.

内存分配:

* 每执行一次Sprintf转换数字, 需要从stack上分配两次内存, 共消耗了16个byte内存.
* 每执行一次FormatInt或者Iota 则0次内存分配, 0 消耗内存.

> `ok      github.com/goinaction/code/chapter9/listing28   5.560s` 则表示运行三个基准测试共花了5.560秒.

通过上面的三个测试, 我们了解到在进行int到string转换的时候, 使用FormatInt是最为快捷的一种方式.

感兴趣的小伙伴可以尝试研究string join哪种方式更快:

1. strings.Join(str1, str2, str3)
2. fmt.Sprintf("%s%s%s", str1, str2, str3)
3. 直接相加, str1 + str2 + str3
4. bytes.Buffer 构造string

哈哈 我也记不清是golang哪个版本了, 以前是bytes.Buffer最快, 但我现在用的golang1.10 现在是第三个方案最快.

---
附第9章小结

* 测试功能被内置到 Go 语言中,Go 语言提供了必要的测试工具。 
* go test工具用来运行测试。
* 测试文件总是以_test.go 作为文件名的结尾。 
* 表组测试是利用一个测试函数测试多组值的好办法。 
* 包中的示例代码,既能用于测试,也能用于文档。 
* 基准测试提供了探查代码性能的机制。