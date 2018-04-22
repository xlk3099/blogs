---
title: "go实战读书笔记（五）：Slice 切片"
date: 2018-04-14T11:18:08+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["Go In Action"]
---

在上一篇我们谈了数组, 但数组有个局限, 就是一旦数组被声明后, 大小不可变.
切片就起到了作为动态数组的作用, 可以按需求自动增长缩小.
切片的动态增长通过函数 `append` 来实现, 动态缩小通过对切片的再切片.

# 切片的内部构成
---

切片是一个很小的对象, 切片由三个字段构成:

  1. 指向底层数组的指针.
  2. 切片访问的元素个数. (长度)
  3. 切片允许的容量.

{{% center %}} 1.1 切片构造示意图:
![image](https://user-images.githubusercontent.com/1768412/38764258-59a17532-3fde-11e8-9502-073092ca834c.png)
{{% / center %}}

</br>

# 切片的创建和初始化
---

1. 通过`make` 声明创建

    ```go
    // 创建一个字符串切片, 默认长度为3, 容量为5.
    slice := make([]string, 3, 5)
    ```
    在make函数里面, 第一个参数是切片类型, 第二个是底层数组的初始长度, 第三个是底层数组的容量.

2. 通过切片字面声明创建

    ```go
    // 创建字符串切片, 其长度和容量都是5.
    slice := []string{"Red", "Blue", "Green", "Yellow", "Pink"}
    ```

3. 空切片, 值为 nil

    切片的0值是`nil`, 所以如果想要创建一个nil切片, 创建的时候, 不需要赋值即可.
    
    通过字面创建空切片:

    ```go
    var slice []string
    // or
    slice := make([]string, 0)
    // or
    slice := []string{}
    ```

    {{% center %}} 2.1 空切片结构示意图
    ![image](https://user-images.githubusercontent.com/1768412/38764350-93d4bdf8-3fdf-11e8-90d7-6c402056a25c.png)
    {{% / center %}}
    空切片使用场景:
      * 函数要求返回一个集合, 但错误发生, 可返回空切片.
      * 数据库查询返回0个结果.

</br>

# 使用切片
---

* **赋值**: 
    切片里面赋值跟数组里面的操作一致. e.g.

    ```go
    // 创建一个长度为5的整型切片.
    slice := []int{1,2,3,4,5}
    // 把第二个元素的值改为6.
    slice[1] = 6
    ```
    切片之所以叫切片, 是因为将给定切片或者数组切分就可以创建一个新的切片. e.g.
    ```go
    // 创建一个新的长度为5的整型切片.
    slice := []int{10,20,30,40,50}
    // 基于给定切片, 创建一个新的长度为2, 容量为4的切片.
    newSlice := slice[1,,3]
    ```
     {{% center %}} 3.1 上述两个切片内部结构示意图
     ![image](https://user-images.githubusercontent.com/1768412/38764430-60e6085a-3fe1-11e8-8eba-b85f17c1ae0a.png)
     {{% / center %}}

    **Tips: 计算切片长度和容量.**

    对于底层数组容量是k的切片slice[i:j]来说:
    * 长度: j-i
    * 容量: k-i

    这里要引起注意的是, 我们前面讲过, 切片内部三个数据最主要的就是指向数组的指针. 因为指针的缘故, 所以在这里, 当我们修改任一一个切片中某一个成员时, 其它跟其共享底层数组的切片内部成员也会发生变化.

* **切片增长**:
    切片比起数组最大的优势就是可以动态增长. 切片通过go内置方程 `append` 来实现切片的增长.
    注意:
    1. 当append时, 增加新的元素后, 没有超过原切片的容量, 那新元素会被安置到对应底层数组里.
    2. 当append时, 增加新的元素后, 超过了原切片的容量, 那么go会新建一个容量更大新的切片, 并把原切片的数据拷贝到新切片里, 在添加要增加的新元素.

    第一种情况:
    ```go
    // 创建一个新的长度为5的整型切片.
    slice := []int{10,20,30,40,50}
    // 基于上述切片, 创建一个新的长度为2, 容量为4的切片.
    newSlice := slice[1,,3]

    newSlice = append(newSlice. 60)
    ```
    根据我们之前的容量公式, `newSlice` 的长度是3-1=2, 容量是 5-1 = 4. 所以在append的时候, newSlice 还有足够容量, 因此在增加了元素60之后, slice[3] 的值现在也被更新成了60.
    {{% center %}} 3.2 append之后的切片内部示意图
    ![image](https://user-images.githubusercontent.com/1768412/38764524-5b24aadc-3fe3-11e8-8411-4f56ca6e6277.png)
    {{% / center%}}

    第二种情况:
    ```go
    // 创建一个新的长度为5的整型切片.
    slice := []int{10,20,30,40}
    // 基于上述切片, 创建一个新的长度为2, 容量为4的切片.
    newSlice := append(slice, 50)
    ```
    当append操作结束之后, 因为超过了原切片底层数组的容量. 因此go给newSlice建了一个新的数组, 把原数组的数据拷贝到了新的数组, newSlice的数组指针也指向了新的数组.
    {{%center%}} 3.3 append结束后, newSlice 有了新的底层数组
    ![image](https://user-images.githubusercontent.com/1768412/38764564-0d483ab2-3fe4-11e8-9e3c-50e10487f18c.png)
    {{%/ center %}}
    小伙伴可能会好奇, append如何处理底层数组容量增长: **当切片底层数组的容量小于1000时, 超容时, 总是会成倍的增加容量. 而一旦数组大小超过1000时, 容量每次增长度为之25%.**

    我们再看一下append函数:
    ```go
    func append(s []T, vs ...T) []T
    ```
    发现 append的参数是一个可变类型, 也即是我们再使用append时, 可以添加多个元素.
    ```go
    slice := []int{1,2,3}
    slice = append(slice, 4,5,6)
    ```
    append也可以添加一个slice, 不过最后要加上"...", 表示将第二个slice的全部元素添加到第一个slice里.
    ```go
    slice1 := []int{1,2,3}
    slice2 := []int{4,5,6}

    slice3 := append(slice1, slice2...)

* **创建切片3个索引**: 
    在前面的几个例子, 我们基于已知切片创建新切片时, 都只用了两个索引(开始,结束), 实际上是可以有三个索引可以使用, 分别表示: `开始`, `结束`, `容量`.
    例子
    ```go
    // 创建一个水果切片, 长度跟容量都是5.
    fruits := []string{"apple", "orange", "plum", "banana", "grape"}
    // 基于水果slice 创建一个新的切片.
    slice := fruits[2:3:4]
    ```
    第三个索引的作用主要是用来限制新切片的容量, 给底层数组提供保护, 避免 图3.2 在append时候,修改了公用底层数组.

    新的长度,容量计算公式, 给定slice[i:j:k]
    * 长度: j-i
    * 容量: k-i

* **迭代切片**:
    切片是一个集合, 在go里, 只要数据是个集合, 就可以使用`range` 来进行对集合的元素进行迭代. 前章谈到的数组. 跟之后要讲到的`map`, `channel` 都可以通过range 来进行迭代.
    
    举个🌰:
    ```go
    // Create a slice of integers.
    // Contains a length and capacity of 4 elements.
    slice := []int{10, 20, 30, 40}

    // Iterate over each element and display each value.
    for index, value := range slice {
    fmt.Printf("Index: %d  Value: %d\n", index, value)
    }

    Output:
    Index: 0  Value: 10
    Index: 1  Value: 20
    Index: 2  Value: 30
    Index: 3  Value: 40
    ```
    {{%center%}}4.1 range迭代示意图
    ![image](https://user-images.githubusercontent.com/1768412/38764811-64b42d02-3fe8-11e8-85e3-008fe32ec14e.png)
    {{%/ center%}}
    当然也是可以通过
    
    >for i:=0; i < len(array); i++ {...}
    
    循环进行迭代, 这里就不赘述了.

* **多维切片**: 
    和数组一样, 切片也可以是多维的, 也是通过组合多个切片来构造多维切片.
    ```go
    // Create a slice of a slice of integers.
    slice := [][]int{{10}, {100, 200}}
    ```

* **在函数之间传递切片**:
    跟数组不同, 前面我们已经分析了切片的构成. 由三个数据结构, 一个指针(8个byte), 两个整型(64位上是8个byte). 也即是总共24个byte, 考虑到在函数间传递24个byte是非常快速的, 因此没有必要在函数传递切片的时候, 采用指针类型. 而且函数间值传递, 复制的只是切片数据本身, 底层的数组并没有被复制.

    事实上 go 在处理pass by value, 跟 pass by reference 的时候, 处理起来前者更高效.

    这里吐槽下 `protobuf`, 在声明proto文件之后, 转换成对应的go文件, `repeated`的数据类型对应的`slice`, 其传递方式用的是指针传递而不是值传递.