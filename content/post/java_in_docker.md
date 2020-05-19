---
title: "Docker 中 java 应用内存配置问题"
date: 2019-10-12T18:17:20+08:00
lastmod: 2019-10-12T18:17:20+08:00
draft: false
keywords: ["java","docker"]
description: "Docker 中 java 应用内存配置解决办法"
tags: ["docker"]
categories: ["docker"]
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
使用 Dockerhub `java:8` 为基础镜像创建 Springboot APP 时，会遇到很多内存配置的问题。首先容器内存使用 cgroup 技术隔离，当对容器作出限制时，低版本 java 找不到正确的内存配置，
而 java8 默认会取总内存 1/4 作为堆内存配置来启动应用，可能会导致 OOM 频繁发生，并且不便于节约服务器资源。

<!--more-->



### 通过环境变量修改内存配置

旧的配置是前运维可能基于之前网上流传比较多的容器内 java 内存配置办法，通过环境变量声明 jvm 参数限制内存，Dockerfile 配置如下：

```dockerfile
FROM java:8

ENV JAVA_OPTS="\
-server \
-Xmx1.5g \
-Xms1.5g \
-Xss1m \
-Xmn1g"

... 

```

实际测试发现，环境变量没有正确影响 java 启动后内存使用。

由于未配置内存参数情况下 java 会根据获取到的系统内存大小的 1/4 作为最大运行内存来启动。这里的错误配置导致了 app 内存使用不可控，不同程序启动占用内存大小不一。有些可能被限制了发挥，而有些则占用了过多的资源导致了浪费。

搜索发现，springboot 这种内置 tomcat 启动读取环境变量要使用 `JAVA_TOOL_OPTIONS`, 这是 java 应用优先级最高的 jvm 参数环境变量。



### 新的配置办法

记得之前看过一篇罗列新版 Java 更新特性的文章（至少不是 Java8），里面写了为了支持容器或者说 Linux cgroup 特性，jvm 会自己检查是否在容器中，可用内存什么的。
按照这个思路，我找到了新版 java8 支持的办法，只需要多添加参数：

```bash
java -XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -jar app.jar
```


### 自动配置内存脚本

下面是从网上找来的一段绍作改动的脚本，利用了容器内 cgroup 给出的值来计算可以使用的内存大小，并设置环境变量，理论上和上面新办法一样的效果

```bash
#!/usr/bin/env bash
RESERVED_MEGABYTES=${RESERVED_MEGABYTES:-256}
limit_in_bytes=$(cat /sys/fs/cgroup/memory/memory.limit_in_bytes)

# If not default limit_in_bytes in cgroup
if [ "$limit_in_bytes" -ne "9223372036854771712" ]
then
    limit_in_megabytes=$(expr $limit_in_bytes \/ 1048576)
    heap_size=$(expr $limit_in_megabytes - $RESERVED_MEGABYTES)
    export JAVA_OPTS="-Xmx${heap_size}m $JAVA_OPTS"
    echo JAVA_OPTS=$JAVA_OPTS
fi

exec catalina.sh run
```

**部分解释**：

默认容器启动不会做内存使用限制（即使用宿主机可用的最大内存，交换分区内存默认为内存两倍），当使用`-m`指定了内存使用量（默认交换分区内存为`1:1`同样大小，可以使用`--memory-swap 1G`来单独指定，`--memory-swappiness=0`可以关闭使用交换分区）。

`9223372036854771712`是64 位系统可计算最大整数值，来源可以参考：[[What is the value for the cgroup's limit_in_bytes if the memory is not restricted/unlimited?](https://unix.stackexchange.com/questions/420906/what-is-the-value-for-the-cgroups-limit-in-bytes-if-the-memory-is-not-restricte)]



### reference

[在 Docker 里跑 Java，趟坑总结](<http://blog.tenxcloud.com/?p=1894>)

[如何设置Docker容器中Java应用的内存限制](<https://yq.aliyun.com/articles/18037>)

[Docker 资源管理探秘：Docker 背后的内核 Cgroups 机制](<https://www.infoq.cn/article/docker-resource-management-cgroups>)

[Docker 资源限制之内存](<https://blog.opskumu.com/docker-memory-limit.html>)

[理解环境变量 JAVA_TOOL_OPTIONS](https://www.jianshu.com/p/be532b5453a6)

##### 延伸阅读

[](https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-memory-6b41e9a955f9)

[9223372036854771712 的解释](https://unix.stackexchange.com/questions/420906/what-is-the-value-for-the-cgroups-limit-in-bytes-if-the-memory-is-not-restricte)


