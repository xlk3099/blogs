---
title: "使用travis 搭建blog的 continuous deployment"
date: 2018-03-19T10:24:55+08:00
draft: false
author: "xlk3099"
categories: ["tech", "tricks"]
tags: ["travis","CI","github"]
---

使用hugo build static blogs 确实很方便，但有个问题，尤其是经常写blog的话。
每次更新一篇日志，都得用`hugo -t [theme]` 重新build 一遍，然后再推送到对应的github.io 的repo。
也就是说，一点小改动，意味着得管理两个repo。。
理想状态其实应该是，新增一篇日志，然后`git push` 完事, blog自动更新，完事。

利用travis ci 搭建CD，下面三个步骤，我们只需要管理日志repo就行。

* 第一步: 去travis网站 enable blog 的repo 搭建

* 第二步: 在travis 对应的blog repo 设置 github 的 api_token

* 第三步：在blog repo里添加对应的 .travis.yml


```yml
sudo: false

language: go

git:
  depth: 1

install: go get -v github.com/spf13/hugo
script:
  - hugo -t even

deploy:
  provider: pages
  skip_cleanup: true
  # token is set in travis-ci.org dashboard
  github_token: $GITHUB_API_KEY
  on:
    branch: master
  local_dir: public
  repo: xlk3099/xlk3099.github.io
  target_branch: master
  # email: deploy@travis-ci.org
  # name: deployment-bot
```