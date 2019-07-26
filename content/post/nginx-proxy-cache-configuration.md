---
title: "nginx 配置 proxy_cache 缓存功能，附带清理缓存脚本"
date: 2019-05-09T2:32:34
lastmod: 2019-05-09T06:51:22
draft: false
keywords: ["nginx","缓存"]
description: "nginx 缓存 proxy_cache 配置办法，及手动挡清理缓存脚本分享"
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
nginx 缓存 proxy_cache 配置可以减轻后端服务接口压力，提升访问速度，甚至可以当作 CDN 服务器来使用。文中略去概念性内容，简单粗暴的只记录关键配置办法，附送手动清理缓存脚本，开袋即食。
<!--more-->

#### 1. 添加 `proxy_cache` 缓存位置等配置
在 `nginx/conf.d` 目录下创建新的配置文件 `cache.conf` 内容如下：

```nginx
# global proxy_cache configuration
proxy_temp_path /tmp/nginx_temp;
proxy_cache_path /tmp/nginx_cache levels=1:2 keys_zone=cache_zone1:50m inactive=1d max_size=1g;
proxy_connect_timeout 5;
proxy_read_timeout 60;
proxy_send_timeout 10;
proxy_buffer_size 16k;
proxy_buffers 4 64k;
proxy_busy_buffers_size 128k;
proxy_temp_file_write_size 128k;
proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_404;
```

#### 2. 在 `location` 模块中配置启用 `proxy_cache`

启用 `proxy_cache` 反向代理配置参考：

```nginx
location ~ .*\.(gif|jpg|png|html|htm|css|js|ico|swf|pdf)$ {
  	proxy_redirect off;
    proxy_set_header         Host $host;
    proxy_set_header         X-real-ip $remote_addr;
    proxy_set_header         X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_pass               http://somewhere;

    proxy_buffering          on; # 避免全局配置关闭代理缓存
    proxy_cache              cache_zone1;   # 定义的 cache_zone 名称
    proxy_cache_key          "$host$request_uri";   # 定义缓存存放目录的子目录命名规则
    add_header Cache         "$upstream_cache_status"; # 增加缓存头信息，便于判断是否命中
    proxy_cache_valid        200 304 301 302 2h;
    proxy_cache_valid        404 1m;
    proxy_cache_valid        any 5m;
}
```

#### 3. 缓存存储生成文件解释

经由以上配置设置后，测试访问会在缓存目录生成两个目录 `nginx_temp`，`nginx_cache`。前一个是临时缓存目录，第二个是落地在磁盘的缓存。

理解落地在磁盘中的缓存目录结构生成方式可以帮助我们做一些额外的事情，比如手动清除目标地址缓存。

查看内容：

```bash
root@8b330add4313:/tmp# head nginx_cache/5/91/36b84c7d2e40d482a****1aba75d7915
f�\����������\k���~$
KEY: [ 此处和谐掉 uri ]
HTTP/1.1 200
X-Application-Context: eurekaserver:test:9099
Date: Thu, 09 May 2019 02:56:38 GMT
Content-Type: application/json;charset=UTF-8
Connection: close

{"tree":{"children":[{"children":[],"name":"sadsad"},{"children":[],"name":"adasdasd"}],"name":"dasdasd"},"billing_address":{"street_address":"随风倒十分","city":"士大夫士大夫","state":"士大夫士大夫"},"shipping_address":{"street_address":"打发士大夫","city":"十分士大夫","state":"士大夫胜多负少"}}
  ... ...
```

其中文件名 `36b84c7d2e40d482a****1aba75d7915` 是记录中 KEY 值的 MD5 码，`/5/91/` 中前面的 “5” 是 MD5 码最后一位，“91” 则是 倒数第 三二 位。

#### 4. nginx `proxy_cache` 利用脚本手动清理缓存
由于某些需求，我们可能需要手动把 nginx 生成的缓存文件删除（据说为了减小磁盘 io 压力，nginx 只会根据设置的过期时间标记缓存过期，而文件还会继续占据空间），根据上一节给出的缓存文件生成原理， [参考博文](https://wuyanteng.github.io/2017/09/17/%E4%BD%BF%E7%94%A8Nginx%E5%8F%8D%E5%90%91%E4%BB%A3%E7%90%86%E5%81%9Acache%E7%BC%93%E5%AD%98-%E5%AE%9E%E7%8E%B0CDN%E5%8A%9F%E8%83%BD/) 中提供的一个清除缓存的脚本，稍作精简，成品如下：

```bash
#!/usr/bin/env sh

cache_purge(){
while read PURGE_URL
do
    URL_NAME=$(echo -n $PURGE_URL | md5sum | tr -dc 0-z)
    FILE_NAME=$(awk '{print "/tmp/nginx_cache/"substr($0,length($0),1)"/"substr($0,length($0)-2,2)"/"$0}' <<<$URL_NAME)
    rm -rf $FILE_NAME && echo "$PURGE_URL deleted!"
done
}

main (){
    if [ "$#" -ne 1 ];then
      echo $"Usage: $0 < url_file | 'url' >"
      exit 1
    fi
    if [ -f $1 ];then
      cache_purge <$1
    else
      cache_purge <<<$1
    fi
}

main $1
```

使用方法很简单，脚本添加执行权限，后面跟上一个参数，可以为需要的清理 `uri` ，或者内容是一行一个 `uri` 列表的文件。

除了脚本的办法，还有很多网友推荐的使用第三方扩展模块 [ngx_cache_purge](https://www.lvtao.net/web/nginx-cache-purge.html
) 来管理删除缓存，缺点需要额外编译 nginx ，今后有必要使用的话会制作 Docker 镜像。

#### Reference
[使用Nginx反向代理做cache缓存-实现CDN功能](https://wuyanteng.github.io/2017/09/17/%E4%BD%BF%E7%94%A8Nginx%E5%8F%8D%E5%90%91%E4%BB%A3%E7%90%86%E5%81%9Acache%E7%BC%93%E5%AD%98-%E5%AE%9E%E7%8E%B0CDN%E5%8A%9F%E8%83%BD/)
