---
title: "golang: Slice tricks 切片技巧"
date: 2018-04-14T15:18:08+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["tricks"]
---

好吧, 看到标题, 可能会想为什么有`slice tricks` 但没有 `map tricks`? 事实上map 就是用来储存查询,K/V pair在语言里也没啥别的用途了. 但slice 不一样, slice可以用来搭建更多的数据结构. 比如stack & queue.

本文参考自 [SliceTricks](https://github.com/golang/go/wiki/SliceTricks)

</br>

# 常用技巧
---

## Append a new slice
  ```go
a = append(a, b...)
  ```

## Copy

```go
b = make([[]T, len(a))
copy(b,a)
// or
b = append([]T(nil), a...)
```

## Cut

```go
a = append(a[:i],a[j:]...)
```

## Delete

```go
a = append(a[:i], a[i+1:]...)
// or
a = a[:i+copy(a[i:],a[i+1:])]
```

## Delete without preserving order

```go
a[i] = a[len(a)-1]
a = a[:len(a)-1]
```

**NOTE** 加入切片里的数据类型是指针或者是一个带有指针元素的数据结构, 再删除数据之后, 我们需要进行GC处理. 那么上面的Cut跟Delete可能会导致内存溢出问题: 因为其它的一些数据可能还在引用a, 所以被a删掉的数据不能被GC. 使用下面代码能解决这些问题.(说白了就是需要,手动GC..)

> Cut

```go
copy(a[i:], a[j:])
for k, n := len(a)-j+i, len(a); k < n; k++ {
    a[k] = nil // or the zero value of T
}
a = a[:len(a)-j+i]

```

> Delete

```go
copy(a[i:], a[i+1:])
a[len(a)-1] = nil // or the zero value of T
a = a[:len(a)-1]
```

> Delete without preserving order

```go
a[i] = a[len(a)-1]
a[len(a)-1] = nil
a = a[:len(a)-1]
```

## Expand

```go
a = append(a[:i], append(make([]T, j), a[i:]...)...)
```

## Extend

```go
a = append(a, make([]T, j)...)
```

## Insert

```go
a = append(a[:i], append([]T{x}, a[i:]...)...)
```

> Insert 进阶:

```go
s = append(s, 0)
copy(s[i+1:], s[i:])
s[i] = x
```

## InsertVector

```go
a = append(a[:i], append(b, a[i:]...)...)
```

## Pop/Shift : slice 这样就具备了 queue的功能

```go
a = append(a[:i], append(b, a[i:]...)...)
```

## Pop Back

```go
x, a = a[len(a)-1], a[:len(a)-1]
```

## Push Back/RPUSH

```go
a = append(a, x)
```

## Push Front/Unshift/LPUSH

```go
a = append([]T{x}, a...)
```

</br>

# 高阶技巧
---

## Filtering without allocation

利用了切片底层数组可以共享

```go
b := a[:0]
for _, x := range a {
    if f(x) {
        b = append(b, x)
    }
}

```

## Reversing

To replace the contents of a slice with same elements but in reverse order. 亮点是只用了`len(a)/2`

```go
for i := len(a)/2-1; i >= 0; i-- {
    opp := len(a)-1-i
    a[i], a[opp] = a[opp], a[i]
}
```

## Shuffling

Fisher-Yates algorithm:

```go
for i := len(a) - 1; i > 0; i-- {
    j := rand.Intn(i + 1)
    a[i], a[j] = a[j], a[i]
}
```