---
title: "一个查找 Docker 镜像包含 tag 的脚本"
date: 2019-06-11T06:40:50Z
lastmod: 2019-06-11T06:47:22Z
draft: false
keywords: ["shell script","docker"]
description: "一个查找 Docker 镜像包含 tag 的脚本"
tags: ["Linux",Docker]
categories: ["Docker"]
author: "Fimreal"

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: flase
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

用 cURL 简单写一个查询 Docker 镜像的脚本，可以非常方便的查看搜索可以选 tag 。

<!--more-->

脚本内容很简单，主要是使用 cURL 从 `registry.hub.docker.com` 获取镜像的 tag 信息，使用 `jq` 工具格式化取出 `tag` 名称。

为了兼容某些没有预装 `jq` 命令的 Linux ，留下一种略复杂的方式来格式化取到内容。

可以高亮过滤 `tag` 内容，例如：

![](https://od.epurs.com/images/2019/06/11/qrJtodTdzt/docker_tags.png)


```bash
#!/usr/bin/env sh

# print usage

if [ $# -eq 0 ]; then
    echo " Usage:
    $0 <docker_images_name> [string_in_tag]

    e.g.  $0 nginx stable"

    exit 1
fi

# get tags

# tags='curl -s https://registry.hub.docker.com/v1/repositories/centos/tags | tr "}]" "\n\b"  | cut -d \" -f 8'
tags='curl -s https://registry.hub.docker.com/v1/repositories/$1/tags |jq -r ".[] | .name" 2>/dev/null'

# filter

if [ -n "$2" ]; then
    eval $tags |grep --color=auto "$2"
else
    eval $tags
fi
```
