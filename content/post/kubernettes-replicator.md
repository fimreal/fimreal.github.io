---
title: "使用 Kubernettes-Replicator 在不同命名空间同步 secret"
date: 2021-11-03T15:46:49+08:00
lastmod: 2021-11-03T15:46:49+08:00
draft: false
keywords: ["kubernetes","helm"]
description: "使用 Kubernettes-Replicator 在不同命名空间同步 secret"
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

kubernetes 最佳实践会教我们创建很多命名空间（namespace）来做服务隔离，但同时服务对应的 configmap 和 secret 也会因为命名空间被分割开。当我们需要对多个命名空间实则相同的对象，执行更新的配置时，同步配置着实有些麻烦。本文通过引入 kubernetes-replicator 来解决这个麻烦的问题。

<!--more-->

## kubernetes-replicator 有什么用？

如 github 仓库简单的介绍，创建一个自定义的 `controller`，保证 `secret` 和 `configmaps` 在多个命名空间可用。

> This repository contains a custom Kubernetes controller that can be used to make secrets and config maps available in multiple namespaces.

实际上它还可以复制 `roles`和`rolebindings`，并且可以通过注解配置是否允许被复制到某个命名空间。当源配置允许被复制时，源目标改变，控制器会自动更新/创建对应对象资源。



## 安装方法

### 1. 可以通过 helm 安装

首先添加 chart repo：

```bash
helm repo add mittwald https://helm.mittwald.de
helm repo update
```

安装：

```bash
helm install kubernetes-replicator mittwald/kubernetes-replicator
```

卸载：

```bash
helm uninstall kubernetes-replicator
```



安装这里可能会遇到镜像 pull 不到的问题，issue 里给出的解决方法是查看 releases 信息确认版本，可以看到有部分 `x.x..0` 版本是没有 `quay.io` 镜像信息的。

解决方式可以通过手动指定版本安装，例如：

```bash
helm upgrade --install kubernetes-replicator mittwald/kubernetes-replicator --version 2.6.3
```

`containerd` 配置国内镜像代理可以使用：

```bash
mirrors:
  "quay.io":
    endpoint:
      - "quay.mirrors.ustc.edu.cn"
      - "quay.azk8s.cn"
```



### 2. 手动安装

对于 yaml 工程师，建议下载下来修改使用，因为里面镜像 tag 是 latest，默认会安装在 `kube-system` 命名空间。

```bash
kubectl apply -f https://raw.githubusercontent.com/mittwald/kubernetes-replicator/master/deploy/rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/mittwald/kubernetes-replicator/master/deploy/deployment.yaml
```



## 常见用法

### 1. 推送复制模式

创建 `secret/configmap/roles/rolebindings` 对象，同时添加 `replicator.v1.mittwald.de/replicate-to` 或者 `replicator.v1.mittwald.de/replicate-to-matching` 注释，配置想要复制到的命名空间。

#### 1.1 通过 ns 名称指定

```yaml
apiVersion: v1
kind: Secret
metadata:
  annotations:
    replicator.v1.mittwald.de/replicate-to: "my-ns-1,namespace-[0-9]*"
data:
  key1: <value>
```



#### 1.2 通过 ns label 指定

```yaml
apiVersion: v1
kind: Secret
metadata:
  annotations:
    replicator.v1.mittwald.de/replicate-to-matching: >
      my-label=value,my-other-label,my-other-label notin (foo,bar)
data:
  key1: <value>
```



#### 1.3 示例

例如要创建 docker 验证 secret ，yaml 配置如下

```yaml
# kubectl create secret docker-registry docker-token --docker-server=registry.epurs.com --docker-username=admin --docker-password=admin123 -oyaml --dry-run=client
apiVersion: v1
data:
  .dockerconfigjson: eyJhdXRocyI6eyJyZWdpc3RyeS5lcHVycy5jb20iOnsidXNlcm5hbWUiOiJhZG1pbiIsInBhc3N3b3JkIjoiYWRtaW4xMjMiLCJhdXRoIjoiWVdSdGFXNDZZV1J0YVc0eE1qTT0ifX19
kind: Secret
metadata:
  creationTimestamp: null
  name: docker-token
type: kubernetes.io/dockerconfigjson
```

添加注释，使其可以同步到其他所有命名空间，修改完如下：

```yaml
apiVersion: v1
data:
  .dockerconfigjson: eyJhdXRocyI6eyJyZWdpc3RyeS5lcHVycy5jb20iOnsidXNlcm5hbWUiOiJhZG1pbiIsInBhc3N3b3JkIjoiYWRtaW4xMjMiLCJhdXRoIjoiWVdSdGFXNDZZV1J0YVc0eE1qTT0ifX19
kind: Secret
metadata:
  creationTimestamp: null
  name: docker-token
  annotations:
    replicator.v1.mittwald.de/replicate-to: ".*"    # 添加注释，正则匹配所有命名空间名称
type: kubernetes.io/dockerconfigjson
```

查看同步复制效果：

```bash
# kubectl apply -f docker-secret.yaml
secret/docker-token created
# kubectl get secret -A |grep docker-token
default           docker-token                                         kubernetes.io/dockerconfigjson        1      9s
kube-system       docker-token                                         kubernetes.io/dockerconfigjson        1      9s
kube-public       docker-token                                         kubernetes.io/dockerconfigjson        1      9s
kube-node-lease   docker-token                                         kubernetes.io/dockerconfigjson        1      9s
infa              docker-token                                         kubernetes.io/dockerconfigjson        1      9s
ingress-nginx     docker-token                                         kubernetes.io/dockerconfigjson        1      9s
netmaker          docker-token                                         kubernetes.io/dockerconfigjson        1      9s
```

⚠️注意：当源对象被删除后，自动复制的对象也会被控制器同步删除。

### 2. 拉取复制模式

#### 2.1 通过 ns 名称指定

注释添加：`replicator.v1.mittwald.de/replication-allowed: "true"` 和 `replicator.v1.mittwald.de/replication-allowed-namespaces` 

```yaml
apiVersion: v1
kind: Secret
metadata:
  annotations:
    replicator.v1.mittwald.de/replication-allowed: "true"
    replicator.v1.mittwald.de/replication-allowed-namespaces: "my-ns-1,namespace-[0-9]*"
data:
  key1: <value>
```

#### 2.2 创建接收对象

注释添加：`replicator.v1.mittwald.de/replicate-from: <namespace>/<objectname>`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-replica
  annotations:
    replicator.v1.mittwald.de/replicate-from: default/some-secret
data: {}
```

#### 2.3 示例

创建 docker 验证 secret ，yaml 配置如下

```yaml
# kubectl create secret docker-registry docker-token --docker-server=registry.epurs.com --docker-username=admin --docker-password=admin123 -oyaml --dry-run=client
apiVersion: v1
data:
  .dockerconfigjson: eyJhdXRocyI6eyJyZWdpc3RyeS5lcHVycy5jb20iOnsidXNlcm5hbWUiOiJhZG1pbiIsInBhc3N3b3JkIjoiYWRtaW4xMjMiLCJhdXRoIjoiWVdSdGFXNDZZV1J0YVc0eE1qTT0ifX19
kind: Secret
metadata:
  creationTimestamp: null
  name: docker-token
type: kubernetes.io/dockerconfigjson
```

添加注释作为源对象，指定允许同步到其他所有命名空间，修改完如下：

```yaml
apiVersion: v1
data:
  .dockerconfigjson: eyJhdXRocyI6eyJyZWdpc3RyeS5lcHVycy5jb20iOnsidXNlcm5hbWUiOiJhZG1pbiIsInBhc3N3b3JkIjoiYWRtaW4xMjMiLCJhdXRoIjoiWVdSdGFXNDZZV1J0YVc0eE1qTT0ifX19
kind: Secret
metadata:
  creationTimestamp: null
  name: docker-token
  annotations:
    replicator.v1.mittwald.de/replication-allowed: "true"
    replicator.v1.mittwald.de/replication-allowed-namespaces: ".*"   # 添加注释，正则匹配所有命名空间名称
type: kubernetes.io/dockerconfigjson
```

准备空 secret，添加注释：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: docker-token-replica
  namespace: kube-system
  annotations:
    replicator.v1.mittwald.de/replicate-from: default/docker-token
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: e30K
```

创建空 secret ，并添加注释。查看创建完内容是否一致：

```bash
# kubectl apply -f docker-secret.yaml -f rep-docker-secret.yaml
secret/docker-token created
secret/docker-token-replica created
# kubectl get -f docker-secret.yaml -f rep-docker-secret.yaml -o yaml
apiVersion: v1
items:
- apiVersion: v1
  data:
    .dockerconfigjson: eyJhdXRocyI6eyJyZWdpc3RyeS5lcHVycy5jb20iOnsidXNlcm5hbWUiOiJhZG1pbiIsInBhc3N3b3JkIjoiYWRtaW4xMjMiLCJhdXRoIjoiWVdSdGFXNDZZV1J0YVc0eE1qTT0ifX19
  kind: Secret
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: ...
      replicator.v1.mittwald.de/replication-allowed: "true"
      replicator.v1.mittwald.de/replication-allowed-namespaces: .*
    name: docker-token
    namespace: default
    ...
  type: kubernetes.io/dockerconfigjson
- apiVersion: v1
  data:
    .dockerconfigjson: eyJhdXRocyI6eyJyZWdpc3RyeS5lcHVycy5jb20iOnsidXNlcm5hbWUiOiJhZG1pbiIsInBhc3N3b3JkIjoiYWRtaW4xMjMiLCJhdXRoIjoiWVdSdGFXNDZZV1J0YVc0eE1qTT0ifX19
  kind: Secret
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: ...
      ...
      replicator.v1.mittwald.de/replicate-from: default/docker-token
      replicator.v1.mittwald.de/replicated-keys: .dockerconfigjson
    name: docker-token-replica
    namespace: kube-system
    ...
  type: kubernetes.io/dockerconfigjson
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

