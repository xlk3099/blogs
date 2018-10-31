---
title: "golang slice cut 分析"
date: 2018-10-31T10:37:01+08:00
draft: false
categories: ["tech"]
tags: ["golang"]
---

昨天在刷Leetcode的时候， 碰到一题：[Remove Duplicates from Sorted Array II](https://leetcode.com/problems/remove-duplicates-from-sorted-array-ii/description/), 当时第一反应就是直接用slice的cut技巧，直接移除重复部分。

这道题目提醒了我，用了这么久的append，我从没想过append操作的space cost，之前在[Go In Action:Slice](./go-in-action-slice.md)里面有提到过，slice的结构其实是一个指针，跟两个整型（length跟capcity），指针指向一个array。那么当slice对进行cut的时候，会对底层array进行怎样的操作？是直接在底层array上面进行操作, 还是建了一个新的array（`space cost O(n)`). 虽然直觉应该是前者，还是自己跑下代码更放心。

对于简单的代码，我比较喜欢用`print`方法调试。

**测试代码**
```go
package main

import (
	"fmt"
)

func main() {
	slice := []int{1, 2, 3, 4}
	for i, v := range slice {
		fmt.Println("address", &slice[i], v)
	}
	s := append(slice[:1], slice[2:]...)
	for i, v := range s {
		fmt.Println("address", &s[i], v)
	}
}
```

**Output**
```go
address 0xc0000181c0 1
address 0xc0000181c8 2
address 0xc0000181d0 3
address 0xc0000181d8 4
address 0xc0000181c0 1
address 0xc0000181c8 3
address 0xc0000181d0 4
```

可以清晰的看到，cut操作是直接在底层array上操作的。

顺带看了眼appen的实现方式，有兴趣的小伙伴可以参考下。
* https://github.com/golang/go/blob/master/src/cmd/compile/internal/gc/ssa.go
* https://github.com/golang/go/blob/master/src/runtime/slice.go