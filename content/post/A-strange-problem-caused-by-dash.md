---
title: "记一次 dash 导致脚本执行失败的问题"
date: 2019-08-19T06:33:00Z
lastmod: 2019-08-19T06:33:00Z
draft: false
keywords: ["dash","kettle"]
description: "众所周知，Debian 系系统默认的 shell 是 `dash`，日常使用也很少发现有不同于`bash`完全不能用的地方。但是最近我碰到了 ubuntu 环境下使用 `sh` 执行特定的定时脚本会不生效，罪魁祸首就是它。"
tags: ["Linux shell"]
categories: ["Linux"]
author: "Fimreal"

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: true
autoCollapseToc: false
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
众所周知，Debian 系系统默认的 shell 是 `dash`（[Debian Almquist shell](<https://en.wikipedia.org/wiki/Almquist_shell>)），日常使用也很少发现有不同于 `bash` ，完全不能用的地方。

最近，我在重写开发搞的 kettle 定时任务时发现了奇怪的问题，日志正常打印，看起来所有 job 都正确执行了，但是数据库中数据却没有按预期更新。反复折腾排除后，发现是脚本第一行声明执行命令不同的缘故，也就是 ubuntu 环境下使用 `sh` 执行会导致数据库数据没有正确更新。
<!--more-->

### 起因

最近这段时间，深感于数据收集清洗的脚本太杂乱，准备一起迁移到 Kubernetes 上去，方便管理和查看执行日志，顺带把旧的脚本都翻新一遍。

之前脚本主要内容是利用 Docker 启动 kettle 镜像，同时把执行任务脚本 bind 到相应目录启动执行，kettle 本身在命令行模式下提供了`kitchen.sh` 启动脚本，使用也比较方便。迁移到 Kubernetes 需要开启 cronjob 特性，同时修改脚本内容，去掉启动 Docker 的繁琐配置，除了启动脚本，剩下的一切复杂过程全部隐藏到 crontab.yaml 配置中。

当然，这时候指定 cronjob 启动脚本就成了定时任务的执行入口，由它来间接调用 `kitchen.sh` 执行定时任务，所以这里成了出问题的高发地。

问题暴露的很奇怪，上线定时任务两天后，开发询问我迁移后的定时任务是否正常执行，因为日志显示一切正常，但是数据库中数据没有按照计划更新。这里出现了不符合逻辑的结果，难道是脚本问题导致的？

### 分析

问题出现后，第一时间确认了 Kubernetes 上日志输出情况，结果发现日志和之前完全一致没有出现任何异常信息。而模拟环境，使用`kubectl`创建容器手动执行时，复现了该现象。那么问题应该是启动脚本了吧。

经过我反复比对，逐步修改启动脚本和原始脚本不同点，不停测试后发现原来写脚本我经常使用的`#!/usr/bin/env sh`在 ubuntu 系统中被解析为使用`dash`执行，修改为`#!/usr/bin/env bash`执行成功了！那么很可能是`dash`导致了最终结果出现偏差，而程序本身可能也由于一些不兼容的写法，造成除了最终结果，过程完全一致挑不出毛病的异常。

那么为什么使用`dash`会导致错误出现，而`bash`在预期情况下一切正常？

在搜索了 hitachi 网站后，我得到一些可能的原因：

1. 首先，官方给出的脚本里开头都是 `#!/bin/sh` ，经常接触 shell script 的人肯定知道这是兼顾兼容性的写法，但是不可避免的一些语法差异会导致出现预期之外的错误，比如在 ubuntu 默认使用 `dash` 时，[脚本执行完毕不会检测到异常，总会返回 0](https://jira.pentaho.com/browse/PDI-14658?jql=text%20~%20%22dash%22)，即执行成功。这和我遇到的情形一致，这里使用的 2018 版本还没有修复该问题。

2. 也有 [issue](https://jira.pentaho.com/browse/BISERVER-955?jql=text%20~%20%22dash%22) 明确指出了，`dash` 很多基础性的语法解析都不同，例如：

   ```bash
   root@deac908482a4:/# bash -c 'echo -e "1\n2"'
   1
   2
   root@deac908482a4:/# dash -c 'echo -e "1\n2"'
   -e 1
   2
   root@deac908482a4:/# bash -c 'echo "1\n2"'
   1\n2
   root@deac908482a4:/# dash -c 'echo "1\n2"'
   1
   2
   ```

3. 还有众多例子提到使用默认`sh`可能带来的异常。

### 反思

虽说 shell script 已经很少会用来编写大型程序组成部件，但是我们依然要牢记默认使用 `sh` 带来的可能未预知的异常！

很多在 `bash` 下熟悉的奇淫巧计，可能在 `ash`、`dash`、`zsh`等常见 shell 中都没法正常工作，跨环境写脚本需要测试，再测试，尤其是判断执行，出了错可能都无法第一时间发现。

反思之前改过的清理 nginx_proxy_cache 缓存的脚本，应用在 alpine 容器中会报异常，后来还是使用 go 写了个小程序来解决交互问题，shell 脚本不可靠，应该只在测试通过的“当前”环境下使用。
