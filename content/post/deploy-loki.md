---
title: "k3s 中以微服务方式部署 loki"
date: 2022-06-21T15:26:16+08:00
lastmod: 2022-06-21T15:26:16+08:00
draft: false
keywords: ["loki"]
description: "k3s 中以微服务方式部署 loki"
tags: ["loki"]
categories: ["loki"]
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

loki 安装方式非常多，推荐的方式是按照官方文档说明，使用 helm 安装 loki-stack。

本文初衷是为了部署一套尽量轻量化的日志系统，对 k3s 集群内多实例服务日志进行收集过滤。为了缩小服务粒度让 k3s 羸弱的服务器节点承担的压力尽可能分散，故选择手动配置微服务方式部署 loki。

文中没有考虑过多的高可用性，需要读者自行添加。

<!--more-->

## 1. 使用 helm 安装 loki 组件

这里先创建 monitor 命名空间，按需修改。

```bash
LOKI_NAMESPACE="monitor"
kubectl create ns ${LOKI_NAMESPACE}
```

#### 1.1 添加 Grafana 官方的 helm repo

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

#### 1.2 Grafana

使用 helm 安装 Grafana，这里用 longhorn 自动创建 pvc，也可以改为`persistence.existingClaim=`指定提前创建的 pvc 或者 template。

测试时删除部分配置，不使用持久化存储等，不影响部署体验。

==> [可自定义 chart 配置项点这里查看](https://github.com/grafana/helm-charts/blob/main/charts/grafana/README.md) <==

```bash
# 创建密码
secret="grafana"
kubectl create secret generic "$secret" \
--namespace ${LOKI_NAMESPACE} \
--from-literal=admin-user=admin \
--from-literal=admin-password=12345678

helm upgrade --install grafana \
--namespace=${LOKI_NAMESPACE} \
--set admin.existingSecret=${secret} \
--set persistence.enabled=true \
--set persistence.storageClassName=longhorn \
--set persistence.accessModes={ReadWriteOnce} \
--set persistence.size=500Mi \
grafana/grafana
```

手动配置部署 Grafana 参考：

https://grafana.com/docs/grafana/next/setup-grafana/installation/kubernetes/



可使用以下命令转发页面到本地 http://127.0.0.1:3000 ，验证部署。

```bash
export POD_NAME=$(kubectl get pods --namespace ${LOKI_NAMESPACE} -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace ${LOKI_NAMESPACE} port-forward $POD_NAME 3000
```

如果是按照上面步骤 helm 安装的，可以使用账号/密码： admin/12345678 登陆。忘记密码可使用命令获取：

```bash
kubectl get secret --namespace monitor grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
# 或者进入 POD 获取到环境变量 GF_SECURITY_ADMIN_PASSWORD
```



（可选）创建 Ingress

```yaml
# ---------------  ing -------------- ##
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: monitor
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: "grafana.k.epurs.com"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port:
              number: 80
  tls:
    - secretName: epurs-com
```

#### 1.3 Loki

安装时配置添加上自动删除 92 天前日志。

持久化配置要求存储 10Gi 以上。

```bash
helm upgrade --install loki \
--namespace=${LOKI_NAMESPACE} \
--set persistence.enabled=true \
--set persistence.size=20Gi \
--set persistence.storageClassName=longhorn \
--set config.compactor.retention_enabled=true \
--set config.compactor.retention_delete_delay=2208h \
grafana/loki
```

这里不考虑暴露 Loki 到公网，使用 https 及账号密码连接的情况，如有需要请查阅官方文档说明，配置不算复杂，用的场景不多，这里不做说明。

#### 1.4 Fluent-Bit

Fluent-Bit 是轻量化的 Fluentd，即使相比于 promtail 占用资源也非常少，同时能轻松扩展实现 Fluent 高级配置功能，非常好用。

Grafana helm repo 中已经内置了 Fluent-Bit chart，只需要传入参数即可实现一键启动 DaemonSet。默认会收集宿主机上 `host containers log`三种类型日志。

```bash
helm upgrade --install fluent-bit grafana/fluent-bit \
--namespace=${LOKI_NAMESPACE} \
--set loki.serviceName=loki.${LOKI_NAMESPACE}.svc.cluster.local \
--set loki.servicePort=3100
```

==> [自定义 Fluent Bit 配置项点这里查看](https://github.com/grafana/helm-charts/tree/main/charts/fluent-bit) <==



非 k3s/k8s 内安装可参考：https://grafana.com/docs/loki/latest/clients/fluentbit/

## 2. Grafana 配置 Loki

打开： http://<grafana_url>/datasources/new

选择创建 Loki

![img](https://od.epurs.com/images/2022/06/21/4QpacQz2Si/%E5%88%9B%E5%BB%BAloki.png)

填入地址（替换掉变量）：http://loki.${LOKI_NAMESPACE}.svc.cluster.local:3100

![img](https://od.epurs.com/images/2022/06/21/laVa2Ga8ve/loki%E5%9C%B0%E5%9D%80.png?)

其他不需要改动，点击 Save & test 保存配置。

打开页面：http://<grafana_url>/explore?%22datasource%22:%22Loki%22

即可看到 loki



<details class="lake-collapse"><summary id="ue6e44f40"><span class="ne-text">最后来看下资源占用情况</span></summary><pre data-language="bash" id="smfiQ" class="ne-codeblock language-bash" style="border: 1px solid #e8e8e8; border-radius: 2px; background: #f9f9f9; padding: 16px; font-size: 13px; color: #595959">kubectl top po -nmonitor
NAME                               CPU(cores)   MEMORY(bytes)
fluent-bit-fluent-bit-loki-2rnr4   4m           45Mi
fluent-bit-fluent-bit-loki-bgxzz   13m          17Mi
fluent-bit-fluent-bit-loki-zkdz7   2m           22Mi
grafana-5df8dcb7cd-cfs8s           1m           30Mi
loki-0                             6m           40Mi</pre><p id="u3a7c8218" class="ne-p" style="margin: 0; padding: 0; min-height: 24px"><span class="ne-text">fluent-bit 跑起来好像也没有说的占用那么低，好在不用特别配置，暂时不考虑限制 cpu 用量了。</span></p></details>

## 3. 使用 Loki 查询日志

#### 3.1 通过 Label 对日志源筛选

在 Loki 页面，点击 Log browser 按条件过滤，例如选择使用 app Label 选择，选到需要的 app 名字，点击 Show logs 查看日志。

![img](https://od.epurs.com/images/2022/06/21/7nZtXldBKY/%E6%97%A5%E5%BF%97%E9%80%89%E6%8B%A9.png)

生成查询语法用`{}`括起来。例如：

```bash
{container="traefik",namespace="kube-system"} 
```

查询结果如下图：

![img](https://od.epurs.com/images/2022/06/21/kIpsE1YRFl/%E6%97%A5%E5%BF%97%E6%9F%A5%E8%AF%A2.png)

基础语法使用如下，用“,”隔开多个条件

- = 
- != 不包含
- =~ 正则匹配
- != 

#### 3.2 简单日志查询语法

用在`{}`之后，空格隔开，可以添加多个。

- | = "<string>"	匹配包含字符串的行
- ! = "<string>"	匹配不包含字符串的行
- | ~ "<string>"	匹配符合正则表达式的行
- ! ~ "<string>"	匹配不符合正则表达式的行



例如：

```bash
{container="traefik",namespace="kube-system"} !~ "/ping" !~"machine"
```

![img](https://od.epurs.com/images/2022/06/21/ZInO7Q6528/%E6%97%A5%E5%BF%97%E6%9F%A5%E8%AF%A22.png)
