---
title: "CoreOS 使用 systemd 定时任务启动容器更新 ssl 证书"
date: 2019-09-23T07:23:00
lastmod: 2019-09-23T08:42:00
draft: false
keywords: ["CoreOS","systemd","ssl certificates","Docker"]
description: "CoreOS 使用 systemd 定时任务启动容器更新 ssl 证书"
tags: ["Docker","CoreOS","Linux"]
categories: ["Linux", "CoreOS", "Docker"]
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
如果你跟我一样“喜欢”并使用着 CoreOS 这个专为运行容器设计的系统，对它极精简和苛刻的安全策略着迷的同时，又烦闹于没有 crond 定时服务来管理一些小计划任务，可以参考这篇文章写的一个实例。
使用 systemd 自带的 timer 即定时器功能来替代 crond 管理启动定时任务。
文中给出我自己服务器上使用 Docker 定时更新 Let's Encrypt 证书的 unit 配置文件，及一些简单的管理查看办法供参考。
<!--more-->
### 配置流程

systemd 定时器配置类似于 windows 中常见的 ini 配置文件风格，需要创建一个 timer 配置指定执行的时间，创建同名，或者指定自定义名称的 service unit 来执行。
具体说明可以参考：

https://wiki.archlinux.org/index.php/Systemd/Timers

#### 创建 unit 文件

创建定时任务 `/etc/systemd/system/acme_epurs.timer` 写入如下内容：

```ini
# /etc/systemd/system/acme_epurs.timer
[Unit]
Description=Acme for epurs.com every 2 mouth
After=network-online.target

[Timer]
OnUnitActiveSec=2month
# 同名可以忽略 Unit 配置
# Unit=acme_epurs.service

[Install]
WantedBy=multi-user.target
```

创建启动服务 `/etc/systemd/system/acme_epurs.service` ，写入如下内容：

```ini
# /etc/systemd/system/acme_epurs.service
[Unit]
Description=update ssl certs for epurs.com
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
# Environment=DOMAIN_NAME=epurs.com DNSS=dns_dp DP_Id=* DP_Key=*
EnvironmentFile=-/server/data/certs/epurs.env
# ExecStartPre=/usr/bin/docker rm -f update_ssl_certs
# 更新 neilpang/acme.sh 镜像
ExecStartPre=/usr/bin/docker pull neilpang/acme.sh
ExecStart=/usr/bin/docker run --rm --name update_ssl_certs -v /server/data/certs/:/certs/ -e DP_Id=${DP_Id} -e DP_Key=${DP_Key} neilpang/acme.sh /root/.acme.sh/acme.sh --force --keylength ec-256 --issue --dns $DNSS --key-file /certs/${DOMAIN_NAME}.key --fullchain-file /certs/${DOMAIN_NAME}.crt --dnssleep 15 -d ${DOMAIN_NAME} -d *.${DOMAIN_NAME}
ExecStop=/usr/bin/docker stop update_ssl_certs
ExecStartPost=/usr/bin/docker exec nginx nginx -s reload

[Install]
WantedBy=multi-user.target
```

#### 重载 systemd 配置

重载 `systemd` 并查看新建的定时器

```bash
systemctl daemon-reload
systemctl list-units --type timer
```

### 参考管理命令

查看定时器列表，类似于 `crontab -l`

```bash
systemctl list-units --type timer
```

查看定时器执行情况，其实是查看定时器对应 serive unit 执行日志

```bash
# 第一种常用办法：
systemctl status unitname.service
# 查看最近 10 条日志：
journactl -xe unitname.service
# 查看所有日志：
journactl -u unitname.service
# journactl -fu unitname.service
```



