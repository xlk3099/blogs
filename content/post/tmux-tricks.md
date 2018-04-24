---
title: "Tmux Tricks"
date: 2018-04-24T10:08:11+08:00
draft: false
author: "xlk3099"
categories: ["tmux"]
tags: ["tricks"]
---


关于tmux神器, 这里就不多介绍了, 自行百度或者google...

这篇起备忘录作用.

对于笔者而言, tmux 最大好处就是断开ssh连接, 远程任务可以继续运行.


# tmux 跟session 相关操作.

```shell
tmux                        # 创建一个新的 session, no name
tmux new -s first           # 创建一个新的 session, 名字为 first
tmux ls                     # 查看当前所有创建的 session
tmux a -t  first            # attach 到名称为 first 的 session
tmux kill-session -t first  # 删除名为 first 的 session
tmux kill-server            # 关闭服务器, 关闭所有session

ctrl+b+d                    # detach 断开当前session, session还是会在后台运行.
```


