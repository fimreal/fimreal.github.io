---
title: "Nginx Tcp 代理添加 IP 限制"
date: 2021-10-14T18:20:49+08:00
lastmod: 2021-10-14T18:20:49+08:00
draft: false
keywords: ["nginx","proxy"]
description: ""
tags: ["nginx","proxy"]
categories: ["nginx","proxy"]
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

<!--more-->

可供容器使用的配置文件：

```nginx
events {
    worker_connections  1024;
}


stream {
    log_format proxy '$remote_addr [$time_local] '
             '$protocol $status $bytes_sent $bytes_received '
             '$session_time "$upstream_addr" '
             '"$upstream_bytes_sent" "$upstream_bytes_received" "$upstream_connect_time"';
    access_log /dev/stdout proxy ;

    server {
        listen 1080;
	      proxy_pass 172.17.0.1:7890;
      	allow 172.17.0.1;
      	allow 127.0.0.1;
      # 单独添加管理白名单 IP 的文件
      # include allow_ip.conf;
        deny all;
    }

}
```

启动：

```bash
docker run -d --name px1080 \
--network host \
-v /root/nginx.conf:/usr/local/openresty/nginx/conf/nginx.conf \
openresty/openresty:alpine
```



待补充：

- lua 自主添加白名单 IP

  参考：https://www.jianshu.com/p/3f2fa52dc66a

- 定期删除白名单 IP

