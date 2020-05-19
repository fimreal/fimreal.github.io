---
title: "记一次解决 Linux 登陆慢的问题"
date: 2020-03-28T14:00:14+08:00
lastmod: 2020-03-28T14:00:14+08:00
draft: false
keywords: ["Linux"]
description: "Linux 运行一段时间，登陆变得非常缓慢，连 reboot 命令都会失败，最终原因竟然是..."
tags: ["Linux"]
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

Linux 运行一段时间，登陆变得非常缓慢，好不容易进入 shell，连 reboot 命令都会失败，最终原因竟然是...

<!--more-->

### 事件背景

家里有一台 mini x86 主机，系统用的 Deepin (从宋老板手里淘过来就预装好了，反正都是熟悉的 Linux ，也没必要再装一遍系统)，用 AC86U dmz 功能隔离暴露在公网上，方便偶尔代理到家里做测试。

问题第一次出现在去年年底，发现 ssh 连接登陆异常困难，通过使用 `ssh -v` 可以看到 ssh 连接建立一切正常，但是迟迟不出下文，需要等待1min 以上才进入 shell 。

这时候进入系统，先简单查看 ps、top 都没有找到异常占用资源的进程，加上 last 等各种分析确定了还没有被入侵。sudo 切换到 root 时，又开始了跟 ssh 登陆一样的等待，等待大概要 1 到 2 分钟才能登陆，登陆后查看系统错误日志。

```
# journalctl -xe -perr
11月 04 21:13:42 *** deepin-sync-daemon[6621]: [ERR] bin.go:79: resp body: no token found map[Token:[] Access-Token:[]]
11月 04 21:13:42 *** deepin-sync-daemon[6621]: [ERR] deepinid.go:178: check token info failed: failed to get token info
11月 04 21:13:42 *** deepin-sync-daemon[6621]: [ERR] bin.go:79: resp body: no token found map[Token:[] Access-Token:[]]
11月 04 21:13:42 *** deepin-sync-daemon[6621]: [ERR] deepinid.go:150: check token info failed: failed to get token info
11月 04 21:13:43 *** deepin-sync-daemon[6819]: [CRI] main.go:38: failed to new deepinid: name 'com.deepin.deepinid' has exists
11月 04 21:14:07 *** pulseaudio[6738]: [pulseaudio] bluez5-util.c: GetManagedObjects() failed: org.freedesktop.DBus.Error.NoReply: Did not receive a reply. Possible causes 1月 04 21:14:32 *** daemon/bluetooth[6590]: bluetooth.go:236: Failed to activate service 'org.bluez': timed out
                                                    ->  bluetooth.go:211
                                                    ->  asm_amd64.s:2361
```

 重点在`org.freedesktop.DBus.Error.NoReply: Did not receive a reply`，看起来是 dbus 挂了，Google 之，发现挺多外国小伙伴遇到这个问题，比如：

https://bbs.archlinux.org/viewtopic.php?id=188287

至于遇到的问题，解决的办法看的我一头雾水。

### 重启能解决 90% 的系统问题

既然问题难以解决，主机上也没有重要内容，重启试试吧。

终端敲下 `reboot`，等待... ，`ctrl + c`无事发生，`systemctl reboot`，等待近 10min， `ctrl + c`还是无事发生...

哎，遇到了重启都重启不了的情况。

Google  之，systemd 不重启怎么解决，得到解决办法，使用强制重启命令：

```
sudo systemctl --force --force reboot
```

带上 2 * `--force`就是厉害，命令一敲系统麻溜的断开了连接。

重启后问题解决了！但是问题根本原因在哪呢 ？

### 终于抓到了问题关键

时隔这么久，主机又被我强制重启了两次，这次我终于忍不了了，`journactl -xe `一点点翻日志 ，但是除了 dbus 记录没有任何相关信息。

这次出问题的不光是 ssh 慢了，docker 里一个代理容器也因为 aws  机器被锁定，ssh 隧道卡死杀不掉进程。于是万能的重启大法来了, `systemctl restart docker`！

本来想重启 docker 解决眼前问题，但是却出现了新的的状况，dockerd 重启没起来，手动执行 `dockerd`，出现报错：

```
[system] Connection has not authenticated soon enough, closing it (auth_timeout=30000ms, elapsed: 30002ms)
```

这里很关键，Google 之，在第一条 [github issue](https://github.com/openbmc/openbmc/issues/1432#issuecomment-293054813) 中发现了惊喜。

> **[mdmillerii](https://github.com/mdmillerii)**: you can explore /proc/*/fd to find processes that have lots of open files files (directories with lots of entries, which are symlinks), then find the commands for those processes numbers like was done on [#875](https://github.com/openbmc/openbmc/issues/875) above.

于是

```
ls /proc/*/fd/
```

发现 3862 进程 fd 有非常非常多的文件描述符，查看进程信息

```bash
lsof -p 3862
```

定位到罪魁祸首是 `xrdp`服务，重启之，测试 ssh 连接，一路畅通，流畅的像新重启完一样。

### 问题总结

根本原因在于 `xrdp`服务占用了过多的 fd ，导致 dbus 服务无响应，也最终致使 systemd 本身都不能好好工作，甚至出现想重启都难的问题。

Google 搜索 "xrdp fd"，第一个 github issue 名字就是 [leaks file descriptors](https://github.com/neutrinolabs/xrdp/issues/279) ，查看修复版本是 0.9.13，deepin apt 仓库里自带的版本是 0.9.1 ，看来是坑在用了 2014 年远古版本的 xrdp，并复现了多次已知 bug 。


Linux 中遇到系统组件报错时，只拿一些通用的底层报错很难搜到解决办法，这时候可以启动一些不关健的小服务，测试调用系统服务时出现的报错信息，来定位原因。


