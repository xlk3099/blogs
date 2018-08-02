---
title: "EOS学习笔记 智能合约（二）"
date: 2018-07-31T15:46:53+08:00
draft: fasle
tags: ["EOS"]
---

## Smart Contract 文件格式

`eosiocpp` 可以使得创建新的contract简单化，它会帮忙创建一个新的contract 模板。
包含两个文件`.hpp` 跟`.cpp`文件。

> apply action handler and the EOSIO_ABI macro

1. 任何一个smart contract 都必须实现`apply` action handler。 apply action handler会监听所有的incoming actions并且执行所需的操作。
2. 通过`code`变量来辨别不同的action，apply 调用receiver，code，action三个input 参数来调用所需的函数 

> The EOSIO_ABI macro

为了让开发者专注在应用的实现上，EOS 提供了EOSIO_ABI宏，开发者只需要将`code`跟`action`的名字提供给这个宏即可。

> abi
 
 ABI 是JSON-Based一个智能合约接口文档，其它用户刻意通过读ABI来了解如何调用你定义的智能合约的方法。
 abi刻意通过 `eosiocpp` 文件生成。

```
$ eosiocpp -g ${contract}.abi ${contract}.hpp
```

## DB multi-index API
通过persistence layer，EOS 提供了一系列服务跟API允许开发者保存action，transaction的状态。
Persistence components 包含这些：
1. 储存状态到数据库里的接口
2. 强化的查询能力从数据库查询数据
3. C++ API 给上述的接口
4. C APIs 用来访问核心服务。

EOSIO Multi-Index API 提供了C++到数据库的接口，允许用户基于不同的index跟排序方式来获取数据，
可参考`multi_index.hpp` 在`contracts/eosiolib`文件夹下。

## Account 的命名规范

1. 只能包含 `.abcdefghijklmnopqrstuvwxyz12345.`, `a-z` (lowercase), `1-5`  `.` 
2. 必须以字母开头
3. 必须是12个字符

## Table 命名规范

* 最多12个字母字符

## Struct 命名规范

* 最多12个字母字符

## Symbols
* 必须是大写字母A-Z
* 小于7个字符