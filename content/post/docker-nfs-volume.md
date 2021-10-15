---
title: "Docker 创建 NFS Volume"
date: 2021-10-15T09:22:52+08:00
lastmod: 2021-10-15T09:22:52+08:00
draft: false
keywords: ["docker","nfs"]
description: "Docker 如何使用 NFS Volume，多个容器使用 NFS Volume 的便捷用法"
tags: ["docker","nfs"]
categories: ["docker","nfs"]
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

Docker 如何使用 NFS Volume，多个容器使用 NFS Volume 的便捷用法。

<!--more-->



文中方法适合 docker 17.06 之后版本，使用`docker version` 确认。



在容器创建时使用下面命令直接挂载：

```bash
docker run -it --name nfs-test \
--mount 'type=volume,source=nfsvolume,target=/mnt,volume-driver=local,volume-opt=type=nfs,volume-opt=device=:/rxt68s30,"volume-opt=o=addr=172.21.0.99,rw,vers=3,nolock,proto=tcp,noresvport"' \
alpine ash
```

查看生成的 volume 信息：

```yaml
# docker volume inspect nfsvolume
[
    {
        "CreatedAt": "2021-09-22T14:22:27+08:00",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/nfsvolume/_data",
        "Name": "nfsvolume",
        "Options": {
            "device": ":/rxt68s30",
            "o": "addr=172.21.0.99,rw,vers=3,nolock,proto=tcp,noresvport",
            "type": "nfs"
        },
        "Scope": "local"
    }
]
```

> 这里 volume 详细信息里会有默认的 Mountpoint 地址 [`/var/lib/docker/volumes/nfsvolume/_data`]，容器启动后，在宿主机上该位置也可以访问到 nfs 里挂载的文件内容，可以延伸出更多用法。



参考上面生成信息，来手动创建 nfs volume：

```bash
docker volume create nfsvolume \
-d local \
-o type=nfs \
-o o=addr=172.21.0.99,rw,vers=3,nolock,proto=tcp,noresvport \
-o device=:/rxt68s30
```



启动容器时，映射使用 nfsvolume ：

```bash
docker run --rm -v nfsvolume:/mnt alpine ls /mnt
```



Done！
