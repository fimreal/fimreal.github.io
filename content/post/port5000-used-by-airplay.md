---
title: "解决 MacOS 12 Monterey 端口 5000 被占用"
date: 2021-11-25T10:24:17+08:00
lastmod: 2021-11-25T10:24:17+08:00
draft: false
keywords: ["MacOS"]
description: "解决 MacOS 12 Monterey 端口 5000 被占用的问题"
tags: ["MacOS"]
categories: ["MacOS"]
author: "Fimreal"

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: true
autoCollapseToc: true
postMetaInFooter: false
hiddenFromHomePage: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
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

npm 调试程序默认启动在 :5000 端口，而在近期 MacOS 更新后启动调试总是提示 5000 端口被占用。

<!--more-->



众所周知，MacOS 虽然属于 Unix 系统，但是和 Linux 有很多不同，例如不存在 /proc 目录，netstat 等网络工具有很大差别，致使很多在服务器上很简单的排查，在 MacOS 上有点无从下手。



首先查询端口占用情况可以使用 `lsof` 这个万能命令：

```bash
$ lsof -i:5000
COMMAND    PID      USER   FD   TYPE            DEVICE SIZE/OFF NODE NAME
ControlCe 1640 XXX   39u  IPv4 0x7f1756d40b55d59      0t0  TCP *:commplex-main (LISTEN)
ControlCe 1640 XXX   40u  IPv6 0x7f1757ba5364a81      0t0  TCP *:commplex-main (LISTEN)
```

可以看到 `ControlCe` 这个命令是占用的进程的启动命令。

Google 之，发现原来是新加入的 AirPlay 占用了 5000 端口。

打开系统偏好-共享，点掉“隔空播放接收器”，问题解决。

