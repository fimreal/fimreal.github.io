---
title: "k3s 自动化升级"
date: 2022-02-15T14:01:54+08:00
lastmod: 2022-02-15T14:01:54+08:00
draft: false
keywords: []
description: ""
tags: []
categories: []
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

k3s 自动/手动升级方法及相关操作

<!--more-->



## 1. k3s 使用 CRD 升级

参考：https://docs.rancher.cn/docs/k3s/upgrades/automated/_index

适用于自建 k3s 集群，使用 Rancher 的 [system-upgrad-controller](https://github.com/rancher/system-upgrade-controller) 来管理手动/自动升级。

总共需要完成两步配置：

1. 安装 system-upgrade-controller，会在集群内创建名为 plans.upgrade.cattle.io 的 CRD
2. 配置更新计划。这里可以使用自动发现 release 模式，保证实时更新到最新版本，同时也可以创建升级到指定版本的一次性更新计划。

### 1.1 安装 system-upgrade-controller

联网执行一句话：

```bash
kubectl apply -f https://github.com/rancher/system-upgrade-controller/releases/latest/download/system-upgrade-controller.yaml
```

更新配置存储在 configmap，默认配置如下：

```yaml
apiVersion: v1
data:
  SYSTEM_UPGRADE_CONTROLLER_DEBUG: "false"
  SYSTEM_UPGRADE_CONTROLLER_THREADS: "2"
  SYSTEM_UPGRADE_JOB_ACTIVE_DEADLINE_SECONDS: "900"
  SYSTEM_UPGRADE_JOB_BACKOFF_LIMIT: "99"
  SYSTEM_UPGRADE_JOB_IMAGE_PULL_POLICY: Always
  SYSTEM_UPGRADE_JOB_KUBECTL_IMAGE: rancher/kubectl:v1.18.20
  SYSTEM_UPGRADE_JOB_PRIVILEGED: "true"
  SYSTEM_UPGRADE_JOB_TTL_SECONDS_AFTER_FINISH: "900"
  SYSTEM_UPGRADE_PLAN_POLLING_INTERVAL: 15m
kind: ConfigMap
metadata:
  name: default-controller-env
  namespace: system-upgrade
```



### 1.2 创建更新计划

按照官方文档说法，推荐使用两份计划，分别对应 server(master) 和 agent(worker) 节点的升级。

按照版本升级配置示例：

```yaml
# Server plan
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: server-plan
  namespace: system-upgrade	
spec:
  concurrency: 1		# 指定同时升级的节点个数
  cordon: true			# 升级禁止调度
  nodeSelector:
    matchExpressions:	
    - key: node-role.kubernetes.io/master		# 指定 master 节点
      operator: In
      values:
      - "true"
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/k3s-upgrade
  version: v1.22.6+k3s1		# 指定版本，也可替换为下面地址，自动升级到最新的 stable 版本
# channel: https://update.k3s.io/v1-release/channels/stable
---
# Agent plan
# 同名字段意义同上
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: agent-plan
  namespace: system-upgrade
spec:
  concurrency: 1
  cordon: true
  nodeSelector:
    matchExpressions:
    - key: node-role.kubernetes.io/master
      operator: DoesNotExist
  prepare:
    args:
    - prepare
    - server-plan				# 这里配置意思是等待 server-plan 计划结束后再执行升级 
    image: rancher/k3s-upgrade
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/k3s-upgrade
  version: v1.22.6+k3s1
```

查看升级进度：

```bash
kubectl -n system-upgrade get plans -o yaml
kubectl -n system-upgrade get jobs -o yaml
```





## 2. k3s 停止并重置节点

由于 k3s 自带的高可用特性，k3s 停止时，POD 会继续运行，如果想要完全剔除 k3s ，可以使用自带的脚本 `k3s-killall.sh`，killall 脚本清理容器、k3s 目录和网络组件，同时也删除了 iptables 链和所有相关规则。但是集群数据不会被删除。



## 3. k3s 全新安装/升级

k3s 覆盖安装也同样会更新旧版本，实现升级目的

默认安装命令：

```bash
curl -sfL https://get.k3s.io | sh -
```

国内安装可使用：

```bash
curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -
```

指定 channel 升级，可选 stable、latest、v1.xx，例如 latest

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=latest sh -
```

