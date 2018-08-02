---
title: "Ubuntu下，golang myzsh 快速搭建"
date: 2018-07-25T14:02:00+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["memo"]
---
0. 安装build essential

```
sudo apt-get install build-essential
```

1. 安装oh-my-zsh

```
apt install zsh

sh -c "$(wget https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"

```

2. 安装最新的golang

```
sudo apt-get update
sudo apt-get -y upgrade
wget https://dl.google.com/go/go1.10.3.linux-amd64.tar.gz
sudo tar -xvf go1.10.3.linux-amd64.tar.gz
sudo mv go /usr/local

```

3. 更新golang 环境

```
vi ~/.zshrc

// Add these lines
export GOROOT=/usr/local/go
export GOPATH=/home/lekai/projects
export PATH=$GOPATH/bin:$GOROOT/bin:$PATH
```