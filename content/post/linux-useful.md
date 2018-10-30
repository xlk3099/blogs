---
title: "Linux 指令备忘录"
date: 2018-04-24T10:08:11+08:00
draft: false
author: "xlk3099"
categories: ["tech"]
tags: ["tricks"]
---

Linux 下有用的top monitoring tool：
```
top - original tool
htop - adds support to multicore/cpu
iotop - input/output monitoring
iftop - network monitoring
atop - merges previous elements into a single overview
gtop - fancy visuals of system stats
slabtop – displays a listing of the top caches
```

Linux 下给指定用户读写执行权利：
    sudo setfacl -m u:lekai:rwx -R /usr/local