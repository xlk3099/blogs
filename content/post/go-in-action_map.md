---
title: "go实战读书笔记（六）：Map 映射"
date: 2018-04-14T13:43:58+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["Go In Action"]
---

不喜欢中文名映射, 还是用map吧...

Map 是一种用来存储一系列K/V键值对的数据结构. Map是无序的.

</br>

# Map的内部构成
---

先看一下map的内部结构.
{{%center%}}1.1 map 内部结构示意图
![image](https://user-images.githubusercontent.com/1768412/38764958-b1c8a2a0-3feb-11e8-827c-1bb159ef2982.png)
{{%/ center%}}

map 里包含了一组buckets. 每当你save, get, 或者delete 任何一个K/V pair的时候, 第一步就是找到对应的bucket.

寻找bucket是通过hash函数实现的. 把key传给hash函数, 就会生成对应的hash值. 在go里面, 通过hash函数生成的hash值又分成两部分, LOB(LOW ORDER BITS) 跟HOB(HIGH ORDER BITS).

hash函数的目的是为了生成索引, 这个索引会将K/V pair 分布到所有可以用的bucket里. 一个好的hash函数会将索引分布的很均匀, 索引分布的越均匀, 访问K/V pair的速度就会越快. 打个比方, 在map里储存10000个元素, 我们不会希望每次访问10000个元素才能找到需要的元素, 我们当然希望我们访问的K/V pair次数越少越好. 对于10000个元素的map, 每次查找只需要查找8个键值对才是一个分布的比较好的映射. 也正是因为使用了hash函数的缘故, 所以数据在map里的储存是无序的.

{{%center%}} 1.2 hash函数简单示意图
![image](https://user-images.githubusercontent.com/1768412/44895464-12783e00-ad27-11e8-87f3-550d592f7ef3.png)
{{%/center%}}
图1.2 展示了hash函数的简单示意图, go 里面的hash函数会更复杂些, 但大体差不多. 这里键是字符串, 代表颜色, 通过 hash 函数, 会转换成一个hash值. 这个数值表示了bucket的位置. 在go里面, 生成的hash值的LOB 会被用来确认储存K/V pair的bucket.

再看 图1.1, 下半部分表示了bucket的内部结构. bucket里面也有两个数据结构, 第一个结构是一个数组, 内部储存了键值通过hash函数之后生成的hash值的HOB(高8位). HOB 用来区分存在同一个bucket里的不同K/V Pair的位置.
第二个数据结构是一个字节数组(byte array). 字节数组先依次储存了这个bucket里面的所有的key, 然后储存了这个bucket里面的从所有值. 这种(K/K.../V/V) 而不是(K/V/K/V)的模式是为了减少每个bucket所需的内存.

</br>

# 2. Map的声明和初始化
---
声明一个map变量时, 我们需要定义map的key的类型, 跟value的类型. 跟切片一样, map的声明跟初始化可以通过make或者字面量来实现.

```go
// 声明一个key为字符串, 值为整型的map
dict := make(map[string]int)

// 声明一个key为字符串, 值为整型的map 并初始化两个K/V pair
dict := map[string]string{"Red": "#da1337", "Orange": "#e95a22"}
```
要引起注意的是, map的key可以是内置的原始数据类型比如`int`, `string`. 也可以是用户自己声明的. 但是, **切片**, **函数**, 以及**包含切片的数据结构** 都不能作为map的key, 使用它们作为map的key时会造成编译错误.

不同的是,value可以是任何类型, 上述在map中会引起编译错误的类型也都可以在value中被使用.

</br>

# 3. Map的使用
---

* 给map添加新的K/V pair:
    ```go
    // 建一个新的空的map, 接受的Key type为string, value 类型也为string 
    colors := map[string]string{}

    // 添加红色及其对应编码到map
    colors["Red"] = "#da1337"
    ```
* 空Map 或者`nil` map 声明
    ```go
    // 建一个新的nil的map by declaring the map only
    var colors map[string]string

    // 添加红色及其编码到颜色表里
    colors["Red"] = "#da1337"

    Runtime Error:
    panic: runtime error: assignment to entry in nil map
    ```
* 检查key是否在map里存在
    
    最常使用的方法
    ```go
    // Retrieve the value for the key "Blue".
    value, exists := colors["Blue"]

    // Did this key exist?
    if exists {
        fmt.Println(value)
    }
    ```
    当然, 在go map里, 只要你查询一个key, map总会返回一个值, 无论那个key存在map里与否. 当然在key不存在的时候,返回的是与value 类型对应的0值. 所以也可以通过这种方式来查询给定的map里存不存在这个key.
    ```go
    // Retrieve the value for the key "Blue".
    value := colors["Blue"]

    // Did this key exist?
    if value != "" {
        fmt.Println(value)
    }
    ```
    不过这种方式也有其弊端, 比如一个key其实在给定的map里存在, 但其保存值刚好是value 类型的0值. 那么依据上述代码会让人误会以为这个key值不存在.

* map的迭代
    跟slice 一样, map的迭代也可以通过range来实现.
    ```go
    // Display all the colors in the map.

    for key, value := range colors {
        fmt.Printf("Key: %s  Value: %s\n", key, value)
    }
    ```
    不过跟slice不同的是, 每一个iteration, 现在返回的是key, value 对, 而不是index, value 对.

* map数据的删除

    想要删除map里某个K/V 对的时候, 可以用go 自带的delete函数
    ```go
    // Remove the key/value pair for the key "Coral".
    delete(colors, "Coral")
    ```
* map在函数建的传递

    在函数间传递map, 也不使用指针传递. 跟slice一样, 当你传一个map到一个函数并且对那个map做出修改, 那么所有对这个map引用的数据都会察觉到这个修改.
    
    举个🌰:
    ```go
    func main() {
        // Create a map of colors and color hex codes.
        colors := map[string]string{
        "AliceBlue":   "#f0f8ff",
        "Coral":       "#ff7F50",
        "DarkGray":    "#a9a9a9",
        "ForestGreen": "#228b22",
        }

        // Display all the colors in the map.
        for key, value := range colors {
            fmt.Printf("Key: %s  Value: %s\n", key, value)
        }

        // Call the function to remove the specified key.
        removeColor(colors, "Coral")

        // Display all the colors in the map.
        for key, value := range colors {
            fmt.Printf("Key: %s  Value: %s\n", key, value)
        }
    }

    // removeColor removes keys from the specified map.
    func removeColor(colors map[string]string, key string) {
        delete(colors, key)
    }
    ```

    运行这个程序, 得到的output
    ```shell
    Key: AliceBlue Value: #F0F8FF
    Key: Coral Value: #FF7F50
    Key: DarkGray Value: #A9A9A9
    Key: ForestGreen Value: #228B22

    Key: AliceBlue Value: #F0F8FF
    Key: DarkGray Value: #A9A9A9
    Key: ForestGreen Value: #228B2
    ```

</br>

---
附第4章小结:

* 数组是切片跟map的基本构成模块.
* 在go里面, slice是最常用来处理集合的数据类型, map是K/V 操作最常用的数据类型.
* 内置函数make允许用来创建切片(并设定长度,容量). 切片跟map的字面也可以用来设定初值.
* 切片有容量限制, 但可以通过append来进行扩容.
* map没有任何容量限制.
* 内置函数len可以用来查询切片跟map的长度(包含元素数量).
* 内置函数只对切片有用.
* 通过组合, 可以构建多维数组或者切片. map也能用来构建nested map. 但slice 或者包含slice的数据类型不能作为map的key.
* 在函数间应该使用pass by value 来传递切片或者是map是便捷快速的, 而且不会对它们对应的底层数据进行复制.