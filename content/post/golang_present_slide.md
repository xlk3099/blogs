---
title: "gopher的ppt：slide"
date: 2018-10-30T15:28:18+08:00
draft: false
categories: ["tech"]
tags: ["golang"]
---

slide是随着go语言诞生而产生的一种幻灯片格式。
[官方文档](https://godoc.org/golang.org/x/tools/present#example-Style) 有记录present的各种syntax以及例子，
想了想，还是做个简单的翻译放入自己的博客，日后参考也是更方便点。


### 官方例子

```
Title of document                   ## 演讲主题
Subtitle of document                ## 演讲副标题 （optional）
15:04 2 Jan 2006                    ## 时间      （optional）
Tags: foo, bar, baz                 ## 标签      （optional）
<blank line>                        ## 以下是作者信息部分， 不同作者以空行分开
Author Name                         
Job title, Company
joe@example.com
http://url/
@twitter_name

* Title of slide or section (must have asterisk)     ## 分页

Some Text                                            

** Subsection

- bullets
- more bullets
- a bullet with

*** Sub-subsection

Some More text

  Preformatted text
  is indented (however you like)

Further Text, including invocations like:

.code x.go /^func main/,/^}/
.play y.go
.image image.jpg
.background image.jpg
.iframe http://foo
.link http://foo label
.html file.html
.caption _Gopher_ by [[https://www.instagram.com/reneefrench/][Renée French]]

Again, more text
```

### 划重点
* 以`*` 加文字 表示开始一个新的slide/section， 以文字部分作为该也标题
* `**` 表示 二级区域， `***`表示三级区域
* `-` 用来展示无序列表

#### 字体
* 加粗 `*bold*`
* 斜体 `_italic_`
* 代码style， 添加 ` `` `

#### 函数调用及展示
slide一大特色就内置一系列函数，可以便捷的展示运行代码，添加多媒体，比如
* 通过函数`.code` 显示代码
* 通过函数`.play` 运行代码
* 通过函数`.image` 插入图片
* 通过函数`.background` 添加背景图片

可以通过关键词`OMIT`, 跟`HL` 引用go文件内部代码。
例子：
给定代码test.go包含如下代码
```
tedious_code = boring_function()
// START OMIT
interesting_code = fascinating_function()
// END OMIT
```

```
.code test.go /START OMIT/,/END OMIT/
```
在slide上会显示成：
```
interesting_code = fascinating_function()
```
code 函数支持两个参数
1. `edit`:会生成可编辑文本区域，e.g. `.code -edit test.go`
2. `numbers`: 给演示代码部分添加行数, e.g. `.code -numbers test.go`

`play` 函数跟 `code` 使用方法非常像，它会在slide上添加一个运行的button。

其它的函数比如image，video用法差不多，看一眼就知道怎么调用，就不一一赘述了...

总而言之，身为一个`gopher`， 如果不懂go社区通用的slide模式，那是可耻的。

今后一段时间内所有的share，demo都用slide格式来练手.