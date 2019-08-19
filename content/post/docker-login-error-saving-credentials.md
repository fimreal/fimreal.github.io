---
title: "Docker login 报证书存储错误的解决办法"
date: 2019-04-11T10:05:58Z
lastmod: 2019-04-11T10:05:58Z
draft: false
keywords: ["Docker"]   
description: "Mint Linux 环境下 docker login 出现证书存储错误解决办法"
tags: ["Docker"]
categories: ["容器","Docker"]
author: "Fimreal"

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: true
autoCollapseToc: true
postMetaInFooter: false
hiddenFromHomePage: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
reward: false
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false

# You unlisted posts you might want not want the header or footer to show
hideHeaderAndFooter: false

# You can enable or disable out-of-date content warning for individual post.
# Comment this out to use the global config.
#enableOutdatedInfoWarning: false

flowchartDiagrams:
  enable: false
  options: ""

sequenceDiagrams:
  enable: false
  options: ""

---
docker login 出现错误，提示：`Error saving credentials: error storing credentials - err: exit status 1, out: Cannot autolaunch D-Bus without X11 $DISPLAY `

<!--more-->

### 环境

使用的是 Mint Linux ，容器为 docker-ce 最新版

```bash
$ uname -a
Linux m01 3.10.0-514.el7.x86_64 #1 SMP Tue Nov 22 16:42:41 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
$ docker --version
Docker version 18.09.0, build 4d60db4
```

### 问题出现

今天在 push 容器镜像时，反复提示没有权限，猜测可能是登陆了其他容器账号验证不过。当我 `docker login` 后才发现问题并不简单，错误完整提示：

```bash
$ docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: epurs
Password:
Error saving credentials: error storing credentials - err: exit status 1, out: `Cannot autolaunch D-Bus without X11 $DISPLAY`
```

### 尝试解决

看到  x11 第一反应以为是系统安装了桌面环境有不兼容的地方，遇到问题先 Google 总不会错，果然在前几个结果中就看到了 docker 相关的 [Issue](https://github.com/docker/compose/issues/6023) 。

Issue 中明确说明了只有在安装 docker-compose 时出现了类似报错，下面有回答，出现错误的根本原因是因为默认安装在 X11 环境下验证时使用的 golang-docker-credential-helpers ，但是由于在命令行中可能无法正常工作，要用 docker-credentials-pass 代替。

docker-compose 其实是依照 swarm mode 中 stack 模式启动容器的工具，对我没有特别用处，于是我做了第一次尝试，卸载掉烦人的包：

```bash
sudo apt remove docker-compse
```

嘛，再次执行 `docker login` ，还是报出同样的报错。

### 问题解决

再往下看 Issue，感谢 chriswue 给出的详细回答，他提到这是在 Ubuntu （Mint 同样是基于 Ubuntu 的发行版）下使用 docker 特有的 bug ，而修复办法不需要特意去卸载 `docker-compose` ，只要 "pass" 掉验证步骤。

##### 解决办法：

1. 首先安装 gnupg2  和 pass 包，并生成 gpg2 key （我没有用到生成步骤一样可行）

   ```bash
   sudo apt install gnupg2 pass
   gpg2 --full-generate-key
   ```

2. 查看生成的 key ，使用 `pass` 加载验证

   ```bash
   gpg2 -k
   pass init "whatever key id you have"
   ```

   这里引号中 要填写前面给出的 trustdb.gpg 路径。

做完上述操作后，再使用 `docker login` 就没有问题了。
