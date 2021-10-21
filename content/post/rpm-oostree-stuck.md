---
title: "FCOS rpm-ostree 使用避坑笔记"
date: 2021-10-21T11:47:46+08:00
lastmod: 2021-10-21T11:47:46+08:00
draft: false
keywords: ["rpm-ostree","fcos"]
description: "FCOS rpm-ostree 使用避坑笔记"
tags: ["rpm-ostree","fcos"]
categories: ["rpm-ostree","fcos"]
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

FCOS rpm-ostree 使用避坑笔记，应该会持续更新。

<!--more-->

## 1. yum 源镜像配置

同为 fedora 系统，fcos 也使用 yum 源，只不过把包管理工具从 dnf 变成了具有事务能力的 rpm-ostree。

yum 地址没有动态配置支持，如果在大陆使用，最好选择几个国内大学源保证下载速度。

例如 USTC ：

http://mirrors.ustc.edu.cn/help/fedora.html

适用于刚安装完，备份加修改镜像源地址一键命令：

```
sudo sed -e 's|^metalink=|#metalink=|g' \
         -e 's|^#baseurl=http://download.example/pub/fedora/linux|baseurl=https://mirrors.ustc.edu.cn/fedora|g' \
         -i.bak \
         /etc/yum.repos.d/fedora.repo \
         /etc/yum.repos.d/fedora-modular.repo \
         /etc/yum.repos.d/fedora-updates.repo \
         /etc/yum.repos.d/fedora-updates-modular.repo
```



## 2. 避免使用 archive 仓库

如果习惯`apt update`建立包索引，对于刚转完 fcos 的环境可能会出现下面这种，等待非常久的情况。

```
# rpm-ostree refresh-md
Enabled rpm-md repositories: fedora-cisco-openh264 fedora updates updates-archive
Updating metadata for 'fedora-cisco-openh264'... done
Updating metadata for 'fedora'... done
Updating metadata for 'updates'... done
⠙ Updating metadata for 'updates-archive'  95% [███████████████████░] (0s)
```

正如`fedora-updates-archive.repo`中注释的描述，该仓库存在非常多陈旧的安装包，索引该仓库会消耗非常多的时间，建议不必要不要使用该仓库。

```bash
cat /etc/yum.repos.d/fedora-updates-archive.repo
# This is a repo that contains all the old update packages from the
# Fedora updates yum repository (i.e. the packages that have made it
# to "stable"). This repo is needed for OSTree based systems where users
# may be trying to layer packages on top of a base layer that doesn't
# have the latest stable content. Since base layer content is locked
# the package layering operation will fail unless there are older versions
# of packages available.
#
# This repo is given a high cost in order to prefer the normal Fedora
# yum repositories, which means only packages that can't be found
# elsewhere will be downloaded from here.
...
```

最直接的办法是把该文件排除掉，确定需要的时候再把名字手动改回来

```bash
mv /etc/yum.repos.d/fedora-updates-archive.repo{,.bak}
```



## 3. 安装完 rpm 包，如何不重启系统生效

默认`rpm-ostree`安装完后会提示需要重启系统完成 commit，但还有一个简单的方法来跳过这一步直接使新安装的包生效，执行：

```bash
rpm-ostree ex apply-live
```

在线生效的缺点是，可能出现位置错误，如果在生产环境不确定安装的包影响范围，最好是安排统一的重启时间。



## 4. 安装/更新失败，卡死怎么办

类似状态：

```
# rpm-ostree status
State: busy
AutomaticUpdatesDriver: Zincati
  DriverState: active; trying to stage 34.20210919.3.0 (3 failed deployment attempts)
Transaction: deploy --lock-finalization revision=6334e11901fba7403600522286d817b520eeb15b66b341b392e0c08ed2576b74 --disallow-downgrade
  Initiator: caller :1.1558
```

`journalctl -xeu zincati.service`可以看到 error 信息。

正常来说要使用取消命令，取消这次事务：

```
# rpm-ostree cancel
Cancelling transaction: deploy --lock-finalization revision=6334e11901fba7403600522286d817b520eeb15b66b341b392e0c08ed2576b74 --disallow-downgrade
```

但是我每次尝试的 cancel 操作，经常等了很久都没有成功返回。

fedora 论坛里有人遇到相同问题，提出解决方法可以在 grub 回滚到上个 commit，还有指出可以粗暴的重启服务解决：

```
# sudo systemctl restart rpm-ostreed.service
# rpm-ostree status
State: idle
AutomaticUpdatesDriver: Zincati
  DriverState: active; trying to stage 34.20210919.3.0 (4 failed deployment attempts)
```

https://discussion.fedoraproject.org/t/cant-cancel-rpm-ostree-null-transaction/25174/9
