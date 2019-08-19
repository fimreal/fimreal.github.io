---
title: "CoreOS 添加 bash-completion 命令补全"
date: 2019-04-16T08:13:50Z
lastmod: 2019-04-16T11:47:22Z
draft: false
keywords: ["CoreOS","Linux","bash-completion"]   
description: "CoreOS 添加 bash-completion 自动补全的办法"
tags: ["CoreOS","Linux"]
categories: ["Linux"]
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
CoreOS 系统常用作容器宿主机系统，开发者本着缩小镜像体积提供核心功能的初衷，没有附带上命令补全功能，这可真要逼死 `TAB` 键按个不停随时感觉要飞起来的 `bash shell` 用户。
本文是参考 github/coreos 上 ISSUE 提供的办法来实践测试写出，最终解决方案有些许优化。
<!--more-->
#### **方法说明**
这里要利用 toolbox 小工具安装好 bash-completion 环境，然后把生成的脚本导入 coreos 中。
PS: `toolbox` 是 Coreos 提供的一个用来扩展系统的沙盒环境，默认使用 Fedora 系统。使用 `Ctrl+D` 或者按 `Ctrl+]` 三次可以退出。安装的包都会保存下载供长期使用。

#### **解决过程**
在 shell 中依次执行：
```bash
$ toolbox
# dnf -y install bash-completion wget
# curl  https://raw.githubusercontent.com/docker/cli/master/contrib/completion/bash/docker -o /usr/share/bash-completion/completions/docker
# cp -R /usr/share/bash-completion /media/root/var/
# exit
$ mkdir /etc/bash/bashrc.d/
$ echo "source /var/bash-completion/bash_completion" >> /etc/bash/bashrc.d/bash-completion.sh
```

##### **参考链接**
https://github.com/coreos/bugs/issues/22
