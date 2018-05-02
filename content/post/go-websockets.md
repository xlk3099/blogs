---
title: "Go Websockets"
date: 2018-05-02T14:11:33+08:00
draft: true
---

关于`websocket`, 也算是常见的一种通信方式...关于websocket协议, 小伙伴们可以参考[rfc6455](https://tools.ietf.org/html/rfc6455).

目前go社区最受欢迎的websocket包当属`gorrila/websocket`, 虽然go官方提供了websocket包: `golang.org/x/net/websocket`, 但官方文档自己都推荐使用gorrila.

Focus on:
  - 怎么创建一个websocket server
  - 怎么创建一个websocket client
  - 如何mock一个websocket server for testing.