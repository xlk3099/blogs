---
title: "利用ssh config维护github多个private repo的deploy keys"
date: 2018-11-15T10:50:10+08:00
draft: false
tags: ["CI/CD", "github", "ssh"]
categories: ["tech", "tricks"]
---

有时候会遇上这样的需求：在一个或者多个服务器上，不想通过账号密码拉取存放在 github 上的多个 private repo，典型的例子便是 CI/CD 工具。

CI/CD 自动推送中，很关键的一环就是在服务器上自动拉取最新的项目代码，如果是自搭的 git 服务器，那还比较简单，直接通过 webhook-post-receive。
但现在很多代码是托管在 github 上的，尤其是同时需要拉取多个项目的话，就不是那么方便，比较流行的解决措施是建立一个`公共账户`，这个公共账户可以同时访问多个项目。

还有一种方法是利用 github 的 deploy keys。可以在项目的 settings->deploy keys 区域设置。

举个例子，做集成测试的时候，项目代码，跟集成测试代码通常放在两个 repo 里

```
https://github.com/xlk3099/projecta
https://github.com/xlk3099/projecta-test
```

一般不想通过账号密码拉取 github repo 的方法就是利用 ssh-keygen 生成 ssh key 保存在的 ssh 目录里，上述例子的话，就可以生成两个 keypair。

(projecta.id_rsa， projecta.id_rsa.pub) 跟 (projecta-test.id_rsa， projecta-test.id_rsa.pub)。

一般情况下对于单个 repo，是需要将 key 加入到 ssh-agent：
比如

```
eval "$(ssh-agent -s)"
Agent pid 59566
ssh-add -K ~/.ssh/id_rsa
```

但这里当加入两个 ssh key 之后，通常是第一个被加入的第一个 key 是可以 work，第二个则不行，因为 ssh agent 找不到正确的 key。

好吧，到这里，第一直觉就是多个项目不能 share 一个 deploy key，这个是不被允许的，当尝试往第二个项目添加同样的 deploy key 的时候，
github 会 error out 并告诉你，那个 key 已经被使用中。deploy key 官方文档也有支出，一个 deploy key 仅能被一个 repo 使用。

我们一般按照 github 官方 clone 一个 repo 时，官方给出的 ssh clone url 是这样的：

```
git@github.com/[user]/[project].git
```

参考 github 官方文档，当通过 ssh clone github 的 project 时，我的 ssh config 文件通常是这样的。
我个人不是很喜欢 ssh agent，一般我会使用或者修改`~/.ssh/config`文件。

```
Host github.com
    HostName github.com
    User git
    IdentityFile /home/xlk3099/.ssh/projecta.id_rsa
    IdentitiesOnly yes

Host github.com
    HostName github.com
    User git
    IdentityFile /home/xlk3099/.ssh/projecta-test.id_rsa
    IdentitiesOnly yes
```

我们知道 ssh url 是这样的格式：

```
ssh user@host
```

也就是说对于官方给出的 ssh url 用户都是`git`, host 都是`github.com`

因此如果采用官方提供的 url 方式来添加项目的话，对于多个 github 项目，ssh 只会使用第一个 key，当我们尝试 pull 第二个 project 时，会 error out，说找不到对应的 sshkey。

原因很简单，项目的 host 都一样，ssh 无法判断第二个 project 该用哪个 key。

**问题总结**:按照 github 官方给的 ssh clone url，所有的项目共享一个 host`github.com`, 按这种格式放到 ssh config 里面，ssh 是无法区分不同的 project 该使用哪个 deploy key 的。

也就是说，为了让 ssh 能区分不同的项目，必须修改 ssh config 里的 host。试着给 host 加上 project 名以便可以区分：

```
Host projecta.github.com
    HostName github.com
    User git
    IdentityFile /home/xlk3099/.ssh/projecta.id_rsa
    IdentitiesOnly yes

Host projecta-test.github.com
    HostName github.com
    User git
    IdentityFile /home/xlk3099/.ssh/projecta-test.id_rsa
    IdentitiesOnly yes
```

这里我们将 host 从 github.com 改成了`[repo-name].github.com`, 这样不同的 project 的 host 便不一样了。

要引起注意的是，当我们 clone 或者 pull 项目最新代码时，不能再用官方提供的格式，需要将 ssh url 改成

```
git@[project].github.com/user/[project].git
```

**注**：对于已经 clone 的项目，我们亦需将，项目内的`.git/config`文件从

```
[remote "origin"]
        url = "ssh://git@github.com/xlk3099/projecta.git"
```

修改成

```
[remote "origin"]
        url = "ssh://git@projecta.github.com/xlk3099/projecta.git"
```

不然也是无法 clone。

参考：

1. [managing deploy keys](https://developer.github.com/v3/guides/managing-deploy-keys/)
2. [multiple-deploy-keys-multiple-private-repos-github-ssh-config](https://gist.github.com/gubatron/d96594d982c5043be6d4)
