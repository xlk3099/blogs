---
title: "EOS 学习笔记 智能合约(一）"
date: 2018-07-31T14:36:45+08:00
draft: false
tags: ["EOS"]
---

`contract` 其实说简单了就是在给定一系列的输入，会产生可预期的输出行为的一种协议。比如游戏里的移动，商业协议里面的财产转移等等。

## EOS smart contract 简介

1. EOS的 `contract` 在区块链上注册，在EOSIO 的`nodes`是执行。
2. EOS smart contract 定义了一系列接口：actions, parameters, 数据，以及实现该接口的代码。
3. contract code 会被编译成标准化字节，所以nodes可以执行。
4. blockchain会储存contract产生的transactions，例如 游戏移动，资产转移。
5. 每一个contract必须要有一个Ricardian Contract 来定义binding terms and context（使用规范）

## 编写EOS Smart Contract 需求

1. C/C++ 经验：EOSIO 大量使用WASM（web-assembly)，当前clang/llvm算是最方便讲程序编译成WASM的工具。
2. Linux/Mac OS 开发经验

## 通信模型
1. EOS 的智能合约包含了一系列的`actions`定义跟类型定义
    * action 定义跟实现了contract的支持的行为
    * 类型 定义了该合约需要的内容跟数据结构
    * action 主要在message-based 通信结构里运作
    * client 调用action，将消息发送到nodeos，nodeos将消息分解成实现了该合约的WASM代码，再继续执行action
    * 合约之间可以互相调用，一个合约可以调用另一个合约的action
2. 智能合约的两种通信模型
    * inline 在当前transaction处理的操作成为inline
    * deferred 激活一个新的transaction
3. 合约之间的通信应该是异步的
    * 资源限制算法会负责管理异步通信可能产生的垃圾

### Inline Communication
Inline action 采用的形式是，要求其它actions是作为其本身一部分。可以当成nested transactions。
如果任何一个action 失败，那么会放弃执行剩余的actions，**调用trasaction里面的inline action 不会对外界产生任何通知，无论成败与否**。

### Deferred Communication

Defferred communication 相当于将action 传给一个peer transaction，deferred action 会被安排在之后的transaction执行，根据producer‘s 排序。
**而且deferred action 并不被保证一定会被执行**。

1. 起源transaction， 只能用来判断request 有没有被成功提交，不能判断有没有被成功执行。
2. **一个transaction 是具有cancel一个deferred transaction 能力**。

### Transaction 跟action 的区别

1. action 只是代表单个操作，transaction 是一系列action的集合（多个操作）.
2. contract，account 的交流通信 是基于action的。
3. action 可以被单独发送，也可以 **复和模式** 被一起执行。

> Transaction with one action

```json
{
  "expiration": "2018-04-01T15:20:44",
  "region": 0,
  "ref_block_num": 42580,
  "ref_block_prefix": 3987474256,
  "net_usage_words": 21,
  "kcpu_usage": 1000,
  "delay_sec": 0,
  "context_free_actions": [],
  "actions": [{
      "account": "eosio.token",
      "name": "issue",
      "authorization": [{
          "actor": "eosio",
          "permission": "active"
        }
      ],
      "data": "00000000007015d640420f000000000004454f5300000000046d656d6f"
    }
  ],
  "signatures": [
    ""
  ],
  "context_free_data": []
}
```

> Transaction with multiple actions

```json
{
  "expiration": "...",
  "region": 0,
  "ref_block_num": ...,
  "ref_block_prefix": ...,
  "net_usage_words": ..,
  "kcpu_usage": ..,
  "delay_sec": 0,
  "context_free_actions": [],
  "actions": [{
      "account": "...",
      "name": "...",
      "authorization": [{
          "actor": "...",
          "permission": "..."
        }
      ],
      "data": "..."
    }, {
      "account": "...",
      "name": "...",
      "authorization": [{
          "actor": "...",
          "permission": "..."
        }
      ],
      "data": "..."
    }
  ],
  "signatures": [
    ""
  ],
  "context_free_data": []
}
```

### Context-Free Actions (独立actions)

 1. Action Name 限制：action的类型通常是base32 加密的 int64。2
 2. Transaction 确认：交易一旦确认，会有receipt产生（倒是跟以太坊一样），收到transaction hash 不代表transaction
    已经被确认，只代表节点成功收到交易。
    确认一个交易被执行，需要在交易历史记录里查询该交易的区块包含该交易。
    所有的action必须在transaction里面被执行，一旦transaction fails，那么里面的所有actions都应该被roll back。
 3. 在处理一个action 之前，eosio会给该action分配一个clean的工作内存。这里面会储存该action所需要的变量。每个action的working memory
    都只能该action 访问。actions之间唯一pass 状态的方法是存储到db再从db里获取。
 4. action的副作用：
    * 改变EOSIO DB的state状态
    * 通知当前transaction的接收者
    * 给receiver发送inline action
    * 产生心得deferred transactions
    * 取消现有的deferred transactions

### Transaction 不足之处
每一个transaction必须在30ms之内倍执行。如果一个transaction有多个action，并且执行这些action耗时超过30ms，那么整个transaction就会fail。
在没有并发需求的情况下，可以将action分别写到几个不同的transaction里。
 
