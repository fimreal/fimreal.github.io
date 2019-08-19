---
title: "nginx location 规则使用正则时改写 request_uri 报错"
date: 2019-05-09T06:51:22
lastmod: 2019-05-09T07:51:22
draft: false
keywords: ["nginx",]
description: "在 nginx 中使用正则匹配，同时改写 proxy_pass $request_uri 的小技巧，记住学会了，做各种各样的缓存规则匹配都不怕。"
tags: ["nginx",]
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
今天在配置 nginx 缓存时，遇到一个棘手的问题，开发反馈需要实现缓存特定路径下的数据，并且要能手动清除缓存。缓存配置和清除的办法在前面已经有介绍，但是给出的路径中间有一位会变动，也就是说需要 location 正则匹配区分。
“众所周知”，location 区块匹配正则表达式时，内部 proxy_pass 如果继续配置使用 request_uri 改写功能（即 http://somewhere/path 形式）会报错不可用，那要怎么解决这个需求呢？事实情况并没有这么简单。
<!--more-->


测试使用环境为 nginx-1.15.8 ，近几年的版本应该都一致。

#### 开发的需求
有 uri ：
```uri
https://domain/api/v2/app/.../24/data
```

需要转到后端服务器为：
```uri
http://domain/app/.../24/data
```

其中只需要缓存 `.../24/data` 内容，但是不能对 `.../24`、`.../24/something` 做缓存，数字位 “24” 会随着查询数据不同变化。

那么条件限定死了：

1. 不可能通过数字前面内容来匹配缓存 path
2. 要匹配可变数字，并且数字后面是固定 `/data`
3. `proxy_pass`需要改写 path ，rewrite 不符合需求。

#### 失败的尝试

可能是之前的知识固化了思维，我以为在 `location` 后写正则，再去改写 path ，且不说怎么把把变化的数字传递给 `upstream_addr`，nginx 不是不允许这样做，会直接给出报错吗？

那么如果使用 `if` 来限定缓存的 path 是不是就可以了，遗憾的是 nginx 也会报错。查阅 [nginx.org](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache) 文档说明可以发现，proxy_cache 不能用在 `if` 语句内。

#### 正确的用法

为什么  `rewrite` 规则可以使用 正则，而且用的很好，nginx 却不给 location 添加支持呢？这时候灵光一闪想到可不可以用 `$1` 变量来代替可变的数字，试了再说 ...

修改配置文件

```nginx
  ...
location ~ /api/v2/app/.../([0-9]+)/data {
  	proxy_redirect off;
    proxy_set_header         Host $host;
    proxy_set_header         X-real-ip $remote_addr;
    proxy_set_header         X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_pass               http://domain/app/.../$1/data;

    proxy_buffering          on;# 避免全局配置关闭代理缓存
    proxy_cache              cache_zone1;   # 定义的 cache_zone 名称
    proxy_cache_key          "$host$request_uri";   # 定义缓存存放目录的子目录命名规则
    add_header Cache         "$upstream_cache_status"; # 增加缓存头信息，便于判断是否命中
    proxy_cache_valid        200 304 301 302 2h;
    proxy_cache_valid        404 1m;
    proxy_cache_valid        any 5m;
}
  ...
```

nginx reload !！

结果竟然没有报错，简单添加一个变量的，就实现了原来觉着不可能实现的需求。

#### 后续测试

测试使用正则表达式匹配，后面改写路径为 “/”

```nginx
# cat test.conf
server {
    location ~ /.*  {
        proxy_pass http://epurs.com/;
    }
}
# nginx -t
2019/05/09 17:13:30 [emerg] 7685#7685: "proxy_pass" cannot have URI part in location given by regular expression, or inside named location, or inside "if" statement, or inside "limit_except" block in /etc/nginx/conf.d/test.conf:3
nginx: [emerg] "proxy_pass" cannot have URI part in location given by regular expression, or inside named location, or inside "if" statement, or inside "limit_except" block in /etc/nginx/conf.d/test.conf:3
nginx: configuration file /etc/nginx/nginx.conf test failed
```

报错。

测试使用正则表达式匹配，使用变量匹配 `request_uri`，后面改写路径为 “/$1”

```nginx
# cat test.conf
server {
    location ~ /(.*)  {
        proxy_pass http://epurs.com/$1;
    }
}
# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

可用。

后续多次测试后发现，当使用正则表达式匹配时，使用不存在的变量 `$dontexist`作为改写路径：

```nginx
# nginx -t
2019/05/09 17:31:09 [emerg] 7787#7787: unknown "dontexist" variable
nginx: [emerg] unknown "dontexist" variable
nginx: configuration file /etc/nginx/nginx.conf test failed
```

报错。

但是如果变量是数字表示，即一个空变量，测试是可以通过的。

#### 结论

可见，nginx 也不是不能用正则来匹配规则，网上经常提到 `location` 用法中说到的 正则匹配时，后面不能再用 “http://domain/path” 其实是错误的说法。

真实情况是，如果前面使用了正则表达式匹配出现多种可能路径，那么后面也需要一个对应的变量来保证路径也是可变的。这个变量只要真实存在哪怕为空，只要不是不存在的变量，就可以通过通过 `nginx -t` 检查。

常用的比如 `$1`、`request_uri` 使用起来可以搭配正则表达式实现灵活转发。
当然要注意使用正则时，优先级并不高的问题。
