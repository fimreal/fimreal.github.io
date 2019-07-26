---
title: "nginx 使用 websocket 的配置参数"
date: 2019-04-23T16:36:34
lastmod: 2019-04-23T19:50:57
draft: false
keywords: ["Git"]   
description: "记录 nginx 使用 websocket 添加的配置参数，解决了之前出现访问 502 的问题"
tags: ["nginx","websocket"]
categories: ["web"]
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
网上有很多 nginx 配置 websocket 的文章，要么配置复杂，罗列了很多没有意义的参数配置，要么只是写出几个关键参数无法保证所有访问的兼容性。
这里我也简要记录一次 websocket 配置修改过程，给出一种配置办法。
<!--more-->

在某项目测试环境和生产环境中，都用到了 tengine(nginx) 来反代 `websocket(ws/wss)` 请求，配置文件除了 upstream 不同，其余全部一致。

前些日子为了统一管理环境，便把使用 `websocket` 后端也加到了 k8s 集群内部，本承想 nodePort 访问会接受所有 TCP/UDP 请求，可以和原来 Docker 模式运行的后端一样，兼容 nginx 反代即可。

但事实情况是，`websocket` 连接变的只能和服务器成功连接，后续通讯全部报异常。查看 nginx 日志中出现了很多 502 记录。

Google 之，发现了 github 有相关 [ISSUE](https://github.com/REBELinBLUE/deployer/issues/310)，其中重点有两句配置：

```nginx
# prevents 502 Bad Gateway error
large_client_header_buffers 8 32k;

# prevents 502 bad gateway error
proxy_buffers 8 32k;
proxy_buffer_size 64k;
```

我倒是没看懂这两个参数配置和 `websocket` 有什么直接就联系。

尝试着只添加了后面两个 proxy_buffers 参数，问题便轻松解决，下面是用到的完整配置内容：

```nginx_conf
server{
    ...

    location /api/v1/ws {
        proxy_pass http://backend/ws;

        proxy_http_version 1.1;
        proxy_read_timeout 60s;
        proxy_send_timeout 12s;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Origin xxx;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_set_header X-NginX-Proxy true;
        
      # prevents 502 bad gateway error
        proxy_buffers 8 32k;
        proxy_buffer_size 64k;
    }

    ...
}
```
