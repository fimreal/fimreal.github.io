---
title: "Kubernetes 上部署 rabbitmq statefulset集群"
date: 2020-05-15T13:43:30+08:00
lastmod: 2020-05-15T13:43:30+08:00
draft: false
keywords: ["rabbitmq","kubernetes"]
description: "Kubernetes 上部署 rabbitmq statefulset集群"
tags: ["rabbitmq","kubernetes"]
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
使用 RabbitMQ 3.7.0 开始内置的 `rabbitmq_peer_discovery_k8s` 插件来实现各实例自动发现，最终用 StatefulSet 部署集群。

<!--more-->

### 一、支持 RabbitMQ 在 k8s 内自动发现

官方文档此处说明：https://www.rabbitmq.com/cluster-formation.html#peer-discovery-k8s

集群内部署用到了内置的 `rabbitmq_peer_discovery_k8s` 插件来实现各实例自动发现。RabbitMQ 3.7.0 及之后版本镜像自带有，更旧的版本需要参考文档说明手动安装环境（制作 Docker 镜像）。

首先在 k8s 内创建包含 `get endpoint` 的 RBAC 使 rabbitmq 可以发现其他节点地址，文件参考：

```yaml
# filename: rabbitmq_rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rabbitmq
  namespace: rabbitmq
automountServiceAccountToken: true
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: endpoint-reader
  namespace: rabbitmq
rules:
- apiGroups: [""]
  resources: ["endpoints"]
  verbs: ["get"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: endpoint-reader
  namespace: rabbitmq
subjects:
- kind: ServiceAccount
  name: rabbitmq
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: endpoint-reader
```



### 二、利用 Configmap 添加配置文件

包含两个文件，`enabled_plugins` 指明了初始化时要加载的插件，`rabbitmq.conf` 重要在使用 hostname 发现。

configmap 定义文件参考：

```yaml
# filename: rabbitmq-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rabbitmq-config
  namespace: rabbitmq
data:
  enabled_plugins: | [rabbitmq_management,rabbitmq_peer_discovery_k8s,rabbitmq_shovel,rabbitmq_shovel_management].

  rabbitmq.conf: |
      ## Cluster formation. See https://www.rabbitmq.com/cluster-formation.html to learn more.
      cluster_formation.peer_discovery_backend  = rabbit_peer_discovery_k8s
      cluster_formation.k8s.host = kubernetes.default.svc.cluster.local
      ## Should RabbitMQ node name be computed from the pod's hostname or IP address?
      ## IP addresses are not stable, so using [stable] hostnames is recommended when possible.
      ## Set to "hostname" to use pod hostnames.
      ## When this value is changed, so should the variable used to set the RABBITMQ_NODENAME
      ## environment variable.
      cluster_formation.k8s.address_type = hostname
      ## How often should node cleanup checks run?
      cluster_formation.node_cleanup.interval = 30
      ## Set to false if automatic removal of unknown/absent nodes
      ## is desired. This can be dangerous, see
      ##  * https://www.rabbitmq.com/cluster-formation.html#node-health-checks-and-cleanup
      ##  * https://groups.google.com/forum/#!msg/rabbitmq-users/wuOfzEywHXo/k8z_HWIkBgAJ
      cluster_formation.node_cleanup.only_log_warning = true
      cluster_partition_handling = autoheal
      ## See https://www.rabbitmq.com/ha.html#master-migration-data-locality
      queue_master_locator=min-masters
      ## This is just an example.
      ## This enables remote access for the default user with well known credentials.
      ## Consider deleting the default user and creating a separate user with a set of generated
      ## credentials instead.
      ## Learn more at https://www.rabbitmq.com/access-control.html#loopback-users
      loopback_users.guest = false
      #
      # memory limit
      vm_memory_high_watermark.absolute = 1024M
      # disk usage limit
      disk_free_limit.absolute = 2GB
---
```



### 三、部署 Rabbitmq StatefulSet 集群

首先参考下面 StatefulSet 声明文件 创建 workload，然后等待 rabbitmq 三节点启动完成。

```yaml
kind: Service
apiVersion: v1
metadata:
  namespace: rabbitmq
  name: rabbitmq
  labels:
    app: rabbitmq
spec:
  type: NodePort
  ports:
   - name: http
     protocol: TCP
     port: 15672
     targetPort: 15672
     nodePort: 15672
   - name: amqp
     protocol: TCP
     port: 5672
     targetPort: 5672
     nodePort: 5672
  selector:
    app: rabbitmq
---
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: rabbitmq
  namespace: rabbitmq
spec:
  serviceName: rabbitmq
  replicas: 3
  template:
    metadata:
      labels:
        app: rabbitmq
    # annotations:
      # prometheus.io/path: /metrics    # prometheus 支持
      # prometheus.io/port: "15692"
      # prometheus.io/scrape: "true"
    spec:
      serviceAccountName: rabbitmq
      terminationGracePeriodSeconds: 10
      containers:
      - name: rabbitmq
        image: rabbitmq:3.7.24
        resources:
          limits:
            cpu: 2.0
            memory: 2.0Gi
          requests:
            cpu: 0.5
            memory: 2.0Gi
        volumeMounts:
          - name: config-volume
            mountPath: /etc/rabbitmq
        ports:
          - name: http
            protocol: TCP
            containerPort: 15672
          - name: amqp
            protocol: TCP
            containerPort: 5672
        readinessProbe:
          exec:
            command: ["rabbitmq-diagnostics", "status"]
          initialDelaySeconds: 20
          periodSeconds: 60
          timeoutSeconds: 10
        env:
          - name: MY_POD_NAME
            valueFrom:
              fieldRef:
                apiVersion: v1
                fieldPath: metadata.name
          - name: MY_POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: RABBITMQ_USE_LONGNAME
            value: "true"
          - name: K8S_SERVICE_NAME
            value: rabbitmq
          - name: RABBITMQ_NODENAME
            value: rabbit@$(MY_POD_NAME).$(K8S_SERVICE_NAME).$(MY_POD_NAMESPACE).svc.cluster.local
          - name: K8S_HOSTNAME_SUFFIX
            value: .$(K8S_SERVICE_NAME).$(MY_POD_NAMESPACE).svc.cluster.local
          - name: RABBITMQ_ERLANG_COOKIE
            value: "xxx-rabbitmq"
      volumes:
        - name: config-volume
          configMap:
            name: rabbitmq-config
            items:
            - key: rabbitmq.conf
              path: rabbitmq.conf
            - key: enabled_plugins
              path: enabled_plugins
```

切换为高可用镜像模式，可以进入其中一个 POD，使用下面命令设置镜像策略：

```bash
rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all", "ha-sync-mode":"automatic"}'
```

最终效果如下图，在 dashboard 中可以查看到 ha-all 策略生效。policies 冲可以看到具体策略信息


![](https://od.epurs.com/images/2020/05/15/XmYNb1VBe2/Snipaste_2020-05-15_13-21-01.png)

![](https://od.epurs.com/images/2020/05/15/LzHpl19zgN/Snipaste_2020-05-15_13-23-26.png)


### 四、监控相关(待补充)

RabbitMQ 3.8.0 及之后版本内置 Prometheus 和 Grafana 支持插件，3.7.0 没有适配插件需要更改旧版本插件编译安装。



### 参考链接

https://github.com/rabbitmq/rabbitmq-peer-discovery-k8s/tree/master/examples



