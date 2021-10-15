---
title: "从 CoreOS 迁移到 Fedora CoreOS 之 用 Podman 代替 Docker"
date: 2021-09-10T09:46:32+08:00
lastmod: 2021-09-10T09:46:32+08:00
draft: false
keywords: ["Docker","Podman"]
description: "从 Docker 迁移到 Podman 记录"
tags: ["Docker","Container"]
categories: ["Docker","Container"]
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

之前个人服务器一直在使用 CoreOS 操作系统，但是由于 CoreOS 系统停更太久了，Docker 版本已经跟不上时代步伐，时不时就出现新镜像没法兼容的情况，遂决定迁移至 Fedora CoreOS，同时尝试使用 Podman 替换掉逐步被业界冷落的 Docker。

<!--more-->

## Docker VS Podman

Podman 是 Redhat 推出的容器管理工具，自从 CoreOS 被 Redhat 收购之后，rkt 就被替换为 Podman，和 Docker 并存预制于 CoreOS 系统内。

Podman 和 Docker 最大的区别就在于拉起容器的过程不同。简单来说：

- Docker 拉起容器需要和 docker engine [也就是 `dockerd`] 通信，`dockerd` 接收到信息后会分别用到 `containerd`和`docker-proxy`来依次完成创建容器和网络转发配置，`containerd` 会通过 `containerd-shim` 去调用 OCI [`runC`]，OCI 完成容器创建拉起进程。
- Podman 没有 daemon 进程启动容器会简单很多，主要通过创建 `conmon`进程来初始化容器。每个容器启动都会伴生一个 `conmon` 进程，有点类似于 k8s POD 的 pause 容器，保留该进程可以用来获取容器产生的`stdout`和`stderr`。

## 使用 systemd 增强 Podman 启动服务

REF：

[红帽8的容器教程](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/8/html/building_running_and_managing_containers/porting-containers-to-systemd-using-podman_building-running-and-managing-containers)

[Podman 文档](https://docs.podman.io/en/latest/markdown/podman-run.1.html)

默认 podman 可以管理容器失败重启，但是如果系统发生重启，则不会生效，需要使用 systemd unit 文件来保证服务开机启动。

例如启动容器：

```bash
podman run -d --name cube --restart unless-stopped -p 10.88.0.1:4080:80 registry.cn-beijing.aliyuncs.com/sae-demo/kube:1.0
```

使用 systemd 来管理启动：

```bash
# 生成 systemd-unit 配置
podman generate systemd --name cube > /etc/systemd/system/container-cube.service
# 重载 systemd
systemctl daemon-reload
# 停止 podman 启动的服务
systemctl stop container-cube.service
# 配置开机启动，同时启动容器
systemctl enable container-cube.service --now
# 查看当前服务状态
systemctl status container-cube.service
```

