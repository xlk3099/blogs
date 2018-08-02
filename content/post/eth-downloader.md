---
title: "Eth 源码分析系列: Downloader"
date: 2018-04-17T14:07:26+08:00
draft: true
author: "xlk3099"
categories: ["eth"]
tags: ["eth"]
---

P2P之间的sync是blockchain里面非常重要的一个环节, 了解P2P之间的sync有助于帮助我们了解P2P之间发生了什么, customize部分代码, 调整参数, 更好的开发三方软件. 

这里以通过geth downloader来了解sync的实现. 进入到`go-ethereum/eth/downloader` 目录

```
> tree .
.
├── api.go
├── downloader.go
├── downloader_test.go
├── events.go
├── fakepeer.go
├── metrics.go
├── modes.go
├── peer.go
├── queue.go
├── statesync.go
└── types.go

0 directories, 11 files

```
我们看到该目录下有11个文件, 其中`api.go` 文件是给外部caller查看当前download状态的, `downloader_test.go`, `fakepeer.go` 是测试专用, `metrics.go` 提供了监听接口, 主要关注`downloader.go`.

`downloader.go` 结构
* 全局变量定义
* 通用error 定义
* Downloader 类型声明
* LightChain 跟BlockChain接口
* Downloader 类型 方法

