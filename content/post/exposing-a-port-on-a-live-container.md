---
title: "帮助已经运行的容器暴露端口"
date: 2021-04-30T17:35:23+08:00
lastmod: 2021-04-30T17:35:23+08:00
draft: false
keywords: ["docker","container","socat"]
description: "帮助已经运行的容器暴露端口"
tags: ["Docker","Container","socat"]
categories: ["Docker","Container","socat"]
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

使用Docker时经常会碰到某些容器需要额外打开端口调试的情况，而 docker-proxy 不支持用户在命令行创建新的端口映射，我们需要额外一些小技巧来实现端口转发。

<!--more-->

### 0. 一些准备

关于如何取到容器的 ip ：

```
docker inspect -f {{.NetworkSettings.IPAddress}} ${CONTAINERNAME}
```

### 1. 使用 iptables 转发端口

Docker本身也是使用 [docker-proxy 通过] iptables 创建端口映射，如果清空了 iptables 规则或者 nat 表中没有 Docker 规则链，docker在创建映射端口的容器时会报错提示下面类似的命令。

自定义一条转发，添加在 nat 表 DOCKER 链上：

```
iptables -t nat -A DOCKER -p tcp -d 0/0 --dport 8000 -j DNAT --to-destination ${CONTAINERIP}:8000
```

按理说应该加在 nat 表里 DOCKER-USER 链上，但是发现有环境没有这个链，报错。关于 nat 表 DOCKER 和 DOCKER-USER 链一些说明：https://docs.docker.com/network/iptables/#add-iptables-policies-before-dockers-rules

查看是否已有规则：

```
iptables -t nat nL
```

删除上述添加的规则[**谨慎**]：

也就是把上面 -A 改成 -D

```
iptables -t nat -A DOCKER -p tcp -d 0/0 --dport 8000 -j DNAT --to-destination ${CONTAINERIP}:8000
```

这种办法的缺点是，几乎只适用于 linux 系统，并且需要 sudo 登录到容器所在的宿主机上。

### 2. 使用 sshd 转发端口

在 Docker 宿主机上使用 ssh 转发到主网卡来暴露容器端口，例如使用命令：

```
ssh -NL 8000:localhost:8080 root@127.0.0.1
```

缺点是需要有 ssh 服务和客户端搭配实现。

### 3. 使用 socat 转发端口

这里使用 socat 容器转发，可以随容器进程一起启动[需要设置 restart 参数]

```
docker run -d --name bind-port -p 8000:8000 alpine/socat TCP-LISTEN:8000,fork TCP-CONNECT:${CONTAINERIP}:8000
```

参考：https://hub.docker.com/r/alpine/socat

通过转发 docker 网卡来实现，也需要登录到容器所在的宿主机上来操作。

### 4. 创建使用共享网络空间的容器暴露端口

涉及的 `--link` 说明：https://docs.docker.com/network/links/

有很多方法可以放在容器内转发端口，这里也使用 socat 举例

```
docker run -d --name bind-port -p 8000:8000 --link ${CONTAINERNAME} alpine/socat TCP-LISTEN:8000,fork TCP-CONNECT:${CONTAINERNAME}:8000
```

`-p 8000:8000` 直接换成 `--network host` 更灵活，根据实际情况来。

参考：https://stackoverflow.com/questions/19897743/exposing-a-port-on-a-live-docker-container

较之前的办法要求更容易达成。

利用 alpine/socat 镜像做一个简化版

```
FROM alpine/socat:latest
RUN echo -e '#!/bin/ash\nexec socat tcp-listen:${1},fork,reuseaddr tcp-connect:rs:${1}' >/entrypoint.sh &&\
    chmod u+x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

启动方式：`docker run -d --restart=unless-stopped -p 9080:${NEWPORT} --link ${CONTAINERNAME}:rs mysocat ``${NEWPORT}`

```
]# docker build -t mysocat -f ./mysocat .
...
Successfully tagged mysocat:latest

]# docker run -d --name tsvc nginx
fdace459ca496e06fc58f40c448a790493df1b53c213064eea6a1ac57a22d87e
]# docker run -d --name fp-tsvc --link tsvc:rs -p 9080:80  mysocat 80
7e98a6ef756b09ce1783b6b7b2ce60990d8076aa5978baaeca63bffc7af811c5

]# curl 127.0.0.1:9080 -I
HTTP/1.1 200 OK
...
```

转发 unix-connect ： `socat TCP-LISTEN:3306,reuseaddr,fork UNIX-CONNECT:/var/lib/mysql/mysql.sock`

或者转发 docker.sock [不安全]:

```
docker run -d --name docker-proxy -p 2375:2375 -v /var/run/docker.sock:/var/run/docker.sock alpine/socat TCP-LISTEN:2375,fork UNIX:/var/run/docker.sock
```

### 小结

上面各种办法中，最通用的办法是创建容器，通过 --link 到原容器，使用端口转发工具创建新容器实现暴露端口。

简单的端口转发工具，例如 socat、gost 都不错。