---
title: "LRU golang 实现"
date: 2018-10-27T11:57:27+08:00
categories: ["tech"]
tags: ["leetcode", "data structures", "lru"]
draft: false
---

 今天刷leetcode的时候就做到了LRU实现 链接: https://leetcode.com/problems/lru-cache/description/. 也是巧, 前两天刚发布一篇关于给redis list添加`expire`属性, 其实跟LRU思路非常相似.

LRU 要求
* 实现`Get(key)`: 要求能在O(1) 时间返回, 找不到便返回-1. 
* 实现`Put(key, value)`: 要求O(1)时间返回, 如果cache已经达到capcity, 那么就删除最少使用的K/V pair.

### **思路**
  1. `Get` 方法要求在O(1), 这不摆明是用map么, 但很显然, 一个map不够, 因为光map并不能满足第二个条件.
  2. 需要一个数据结构, 将所有的数据按时间排序.
  3. 每次获取或插入一个键值对, 会从新更新数据结构里的排序, 但这里的顺序更新变化很小, 如果是获取, 在该键值对顺序之前的键值对排序不用变, 之后的都需要`rank-1`.
  4. 每次更新时, 需要记录最少使用的键值对, 因为一旦超过cache的容量, 要将最少使用的键值对删除.
  5. 操作需要达到O(1), 这里不能采用遍历, 结合2,3 就变得很明显了, 需要一个`double linked list`. 

除了double linked list, 还需要额外两个pointer, head跟tail用来分别记录最新查询的键值对跟最久之前查询的键值对, 不然还是要遍历list, 达不到O(1)的操作.

### **实现**

按照上述思路,先实现LRUCache struct, 需要一个map 跟一个double linked list, 跟两个指针, 还有一个capcity 属性用来记录cache是否超出.

```go
type LRUCache struct {
    nodes map[int]*Node
    tail *Node
    head *Node
    capacity int
}

func Constructor(capacity int) LRUCache {
    var lru LRUCache
    lru.capacity = capacity
    lru.nodes = make(map[int]*Node, capacity)
    return lru
}

type Node struct {
    val int
    prev *Node
    next *Node
}
```

#### **Get 方法需要考虑的点**

1. 不存在情况, 按要求返回-1
2. 如果存在, 要给键值对重新排序, 最后返回该值.
    * 如果取值对象是double linked list的头部, 那无需任何操作.
    * 如果取值对象是double linked list的尾部, 需要如何更新顺序.
    * 如果取值对象是double linked list的中间,(一般情况), 需要如何更新顺序.

这样一思考, `Get`方法实现就很直接了, 只需要处理下double linked list的排序操作即可, **这一块一直是我弱项, 每次都要prev, next, head, tail的绕来绕去...甚是麻烦**
```go
func (this *LRUCache) Get(key int) int {
    // if key not exist, return -1
    if _, ok:= this.nodes[key]; !ok {
        return -1
    }
    // if key exist
    this.updateSeq(key)
    return this.nodes[key].val
}

func (this *LRUCache) updateSeq (key int) {
    // if key happens to be the head.
    n := this.nodes[key]
    if this.head == n {
        return
    }
    // if key happens to be the tail
    if this.tail == n  {
        this.head.next = n
        n.prev = this.head
        this.tail = n.next
        this.tail.prev = nil
        this.head = n
        this.head.next = nil
    }else{ // general case
        n.prev.next = n.next
        n.next.prev = n.prev
        this.head.next = n
        n.prev = this.head
        this.head = n
        n.next = nil
    }
    return
}
```
#### **Put方法需要考虑的点**
  1. corner cases: 
    * capcity 为 0的情况.
    * cache 为空, 第一次插入.
  2. 插入对象已经存在时, 我们同样需要像Get一样, 更新顺序
  3. 插入对象不存在时
    * 容量未超出情况
    * 容量超出情况, 我们需要从map里删除tail

为了实现`容量超出情况, 我们需要从map里删除tail`达到O(1)需求, 也就是说我们的tail除了记录value外, 还需要记录key, 那么之前的double linked list数据结构得稍作修改.

```go
type Node struct {
    key int
    val int
    prev *Node
    next *Node
}
```

按照上述思路, 实现Put 方法
```golang
func (this *LRUCache) Put(key int, value int)  {
    // corner case, map is empty
    if this.capacity == 0 {
        return
    }
    // when cache is empty
    if len(this.nodes) == 0 {
        this.nodes[key] = &Node{
            key: key,
            val: value,
            prev: nil,
            next: nil,
        }
        this.head =this.nodes[key]
        this.tail = this.nodes[key]
        return
    }
    // if key exist in the map, just need to update the sequence.
    if _, ok:= this.nodes[key]; ok {
        this.nodes[key].val = value
         this.updateSeq(key)
         return
    }
    // if key not exist in the map
    var n = &Node{}
    n.key = key
    n.val = value
    n.next = nil
    n.prev = this.head
    this.head.next = n
    this.head = n
    // when capacity has been reached, need to delete the tail node.
    if len(this.nodes) == this.capacity {
        delete(this.nodes, this.tail.key)
        this.tail = this.tail.next
        this.tail.prev = nil
    }
    this.nodes[key] = n
    return
}
```

这样就实现了LRU.
附上Leetcode 完整代码.

```go
type LRUCache struct {
    nodes map[int]*Node
    tail *Node
    head *Node
    capacity int
}

type Node struct {
    key int
    val int
    prev *Node
    next *Node
}

func Constructor(capacity int) LRUCache {
    var lru LRUCache
    lru.capacity = capacity
    lru.nodes = make(map[int]*Node, capacity)
    return lru
}

func (this *LRUCache) Get(key int) int {
    // if key not exist, return -1
    if _, ok:= this.nodes[key]; !ok {
        return -1
    }
    // if key exist
    this.updateSeq(key)
    return this.nodes[key].val
}

func (this *LRUCache) updateSeq (key int) {
    // if key happens to be the head.
    n := this.nodes[key]
    if this.head == n {
        return
    }
    // if key happens to be the tail
    if this.tail == n  {
        this.head.next = n
        n.prev = this.head
        this.tail = n.next
        this.tail.prev = nil
        this.head = n
        this.head.next = nil
    }else{ // general case
        n.prev.next = n.next
        n.next.prev = n.prev
        this.head.next = n
        n.prev = this.head
        this.head = n
        n.next = nil
    }
    return
}

func (this *LRUCache) updateSeq (key int) {
    // if key happens to be the head.
    n := this.nodes[key]
    if this.head == n {
        return
    }
    // if key happens to be the tail
    if this.tail == n  {
        this.head.next = n
        n.prev = this.head
        this.tail = n.next
        this.tail.prev = nil
        this.head = n
        this.head.next = nil
    }else{ // general case
        n.prev.next = n.next
        n.next.prev = n.prev
        this.head.next = n
        n.prev = this.head
        this.head = n
        n.next = nil
    }
    return
}
```

