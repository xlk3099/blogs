---
title: "go实战读书笔记（十八）: 标准库 - JSON"
date: 2018-04-18T19:03:13+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["Go In Action"]
---

go经常遇到的一个问题就是JSON解码, 现在很多data serialization 都是JSON格式. 可以说是现在后端跟前端最常用的通信数据格式.
比如REST API啊, 包括JSON RPC, response 都是JSON format. 

**注意:这里我没有参照go-in-action, json 那节** 它JSON encoding decoding的logic讲的不是很清楚. 

这篇文章主要参考自https://medium.com/go-walkthrough/go-walkthrough-encoding-json-package-9681d1d37a8f

<br/>

# JSON 编码
---
JSON package 包提供了两种方式对value进行JSON 编码. 第一个是比较试用于stream data, json.Encoder会将数据编码成io.Writer类型.

```go
type Encoder struct{}

func NewEncoder(w io.Writer) *Encoder

func (enc *Encoder) Encode(v interface{}) error
```

还有一种方式是json.Marshal() 会将你需要的编码的数据变成内存里的byte slice.

```go
func Marshal(v interface{}) ([]byte, error)
```

当数据被传入到JSON encoder的时候, json 库会经过一系列复杂的类型检测, 编译编码器, 并且recursively 处理你的数据. 


1. Type Insepection
当数据被传入到编码器时, json包第一步是检查被传入的数据类型. 数据类型可以通过go 的 reflect包来检查, 而且json包里有一个默认的类型匹配表. 对于go`原始类型`比如int, string, map,  struct, 跟slice. 它们在json包里都有对应的encoder, 比如intEncoder, stringEncoder. 被转换成json类型的时候也相对比较简单, `stringEncoder` 比如会将string值加上双引号以及插入适当的逃逸字符, 以及intEncoder会将整型转换成字符串格式.

2. Encoder 编译
对于非自带类型, go会创建一个encoder. 第一步, encoder会检查该类型有没有实现json包里的接口Marshaler.
```go
type Marshaler interface {
    MarshalJSON() ([]byte, error)
}
```
如果有的话, 那么marshaling 函数会采用该类型自定义的MarshalJSON 类型. 这个方法真的很实用, 尤其是你的自定义类型有特殊的JSON编码格式.

接下来会检查该类型有没有实现TextMarshaler接口.
```go
type TextMrshaler interface {
    MarshalText() (text []byte, err error)
}
```
如果有的话, 那么json包会调用这个方法, 然后再把生成的数据转换成json字符串格式. 比如time.Time 就定义了MarshalText方法.

如果上面两个接口都没有被实现, 那么json包会recursively 会依据原始编码器创建一个encoder. 比如, 一个类型包含了一个struct, 里面有两个字段, 一个整型字段更一个字符串字段, 那么json包会给这个struct生成一个structEncoder, 里面包含了一个intEncoder, 跟stringEncoder. 这里需要指出的是, 生成的该structEncoder只会被build一次, 然后会被保存在内存里.

3. 针对每个字段的Tag option
我们上面讲过, json包里是带有struct encoder的, 关于struct encoder 一个重要的信息是,
他会读取字段的tags来了解encoding时的一些设定. tags 是用 `` ` ` `` 包围的字符串.
比如:
```go
type User struct {
        Name    string `json:"name"`
        Age     int    `json:"age,omitempty"`
        Zipcode int    `json:"zipcode,string"`
}
```
通过tag, 可以做到
* 修改field在json中显示的filed名. 比如在json中, 每个field都是camel格式, 所以保持这种规范是有必要的.
* `omitempty` 可以帮助移除空值. 在json中不显示.
* string 可以帮助强制转换一些非字符串格式成string格式, 比如整型.

4. 递归解析
当编码发生的时候, 所有的值都会被写入到encodeState, 一个内部buffer. 所有类型的encoder 比如intEncoder, stringEncoder都会把转换的值append到encodeState的bytes里.

<br/>

# JSON 解码
---

把json编码的bytes解码成对应的数据类型相当于把上述的过程反转.
有两种方式对JSON数据进行解码, 第一种也是针对stream-based json:
```go
type Decoder struct {}
func NewDecoder(r io.Reader) *Decoder
func (dec *Decoder) Decode(v interface{}) error
```

或者也可以使用json.Unmarshal()函数:
```go
func Unmarshal(data []byte, v interface{}) error
```
解码分两步操作: 
* `scanner` 会tokenize输入数据
* `decodeState` 将那生成的token转成go数据类型.

1. Scanning JSON
scanner是内部一个用来parse json数据的状态机. 第一步, 它会检查第一个byte, 如果第一个byte是`{`, 那么意味着需要解析的是一个object类型, 如果第一个byte是`[`, 那么对应的需要解析的是一个数组. 这也适用于简单数据类型, 如果是双引号开头, 那么标志着字符串的开始, 如果是`t`, 或者`f`, 那表示解析的对象是一个布尔值, 0-9对应了数字的开始.
一旦定下需要解析的类型, 那么scanner会将剩下的工作转到类型相关的函数, 比如string scan, number scan 等等. 对于负责的类型, 比如map或者数组, 会有一个stack用来最终关闭的括号.可能是`]`, `}`.

2. Lookahead buffers
json scanning 一个比较有意思的是lookahead buffer. JSON是`LL(1) parseable`, 意味着每次它只需要一个byte buffer用来scan. 这个buffer会用来瞥一眼下一个byte.
举个🌰, number scanning 函数会一直扫描bytes直到它扫到一个非数字类型的byte. 然后, 那个byte其实已经被读了, 这时候需要将它放回buffer里给下一个scanning function采用.

3. 解码tokens
一旦所有的tokens都被扫描完毕, 它们需要被解析, 被还原. 这个任务交给了decodeState. 在这一个过程中, 每一个token都会被还原成与input值(其实是类型)对应的数据类型.
举个🌰, 假如给Unmarshal第二个参数 是一个struct类型. 那么需要得到的第一个token应该是`{`, 任何其他类型的token都会报错. 整个过程, 将token转换成struct型数据的时候, 会大量的时候reflect package. 跟encoder不一样的是, decoders不会被cache, 所以reflection每一次decode都会被重做一遍.

4. 自定义unmarshaling
跟encoding一样, decoding也支持自定义. 对应的数据类型实现下面两个接口即可:
```go
type Unmarshaler interface {
        UnmarshalJSON([]byte) error
}
```
或者
```go
type TextUnmarshaler interface {
        UnmarshalText(text []byte) error
}
```

__**注意**__: json.Unmarshaler 有个alternative就是 json.RawMessage type. 当一个数据类型被声明成json.RawMessage, 那么这个数据可以一直保存到unmarshaling complete之后再做额外处理. 好处就是可以根据另一个字段的值来进行动态处理.
```go
type T struct {
        Type  string          `json:"type"`
        Value json.RawMessage `json:"value"`
}
func (t *T) Val() (interface{}, error) {
        switch t.Type {
        case "foo":
                // parse "t.Value" as Foo
        case "bar":
                // parse "t.Value" as Bar
        default:
                return nil, errors.New("invalid type")
       }
}
```

还有一个要引起注意的就是json number decoder会默认将数据转换成float64 如果对对应的字段类型是interface{}的话.

<br/>

在go里, 得到json或者将值转成json之后, 对应的json值是一个长串bytes, 没有任何的intent. 读起来会比较麻烦. json包里提供了Indent方法来将其美化 这里就不拓展了.

**结篇:** go的json包提供了非常强大的功能, 基本可以满足所有json encoding/decoding的需求. 我前前后后也碰到了很多json encoding/decoding的问题, 日后可能做一个整理来方便参考.