---
title: "通过创建 svc，在 k8s 内优雅的访问集群外部服务"
date: 2021-10-15T09:49:26+08:00
lastmod: 2021-10-15T09:49:26+08:00
draft: false
keywords: ["kubernetes"]
description: "通过创建 svc，在 k8s 内优雅的访问集群外部服务"
tags: ["kubernetes"]
categories: ["kubernetes"]
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



#### 用 ExternalName 创建通过域名提供的服务

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-ali
spec:
  type: ExternalName
  externalName: rm-bp1xxxxxxxxxxxxxx.mysql.rds.aliyuncs.com
```

`externalName` 也可以填写 IP，但是不推荐。



转发 IP:port 形式，可以使用下面方法



#### 手动创建 EndPoint 来绑定服务

```yaml
# 服务例子
apiVersion: v1
kind: Service
metadata:
  name: registry-gcr
spec:
  type: ClusterIP
  ports:
    - port: 5000
      targetPort: 5000
      
---

# 手动创建 ep，如果已存在服务，需要修改指向外部服务，可以直接编辑 ep 配置，将 subsets 部分修改
apiVersion: v1
kind: Endpoints
metadata:
  name: registry-gcr
subsets:
- addresses:
  - ip: 10.200.0.101		# 集群外部 IP
  ports:
  - port: 5000
  

```

可以通过 `kubectl get ep` 确认前后地址是否符合预期。



#### Ref.

https://kubernetes.io/zh/docs/concepts/services-networking/service/#externalname

https://blog.opskumu.com/kubernetes-ext-service.html
