---
title: "Kubernetes 上部署维护 ProxySQL StatefulSet 集群笔记"
date: 2020-05-14T16:27:36+08:00
lastmod: 2020-05-14T16:27:36+08:00
draft: false
keywords: ["kubernetes","proxysql","MySQL"]
description: "Kubernetes 上部署维护 ProxySQL StatefulSet 集群笔记"
tags: ["kubernetes","proxysql","MySQL"]
categories: ["database"]
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
一些在 k8s 上部署 Proxysql 集群时的的配置记录，
一些在使用 ProxySQL 时遇到的问题，及解决办法。

<!--more-->

### 一、StatefulSet 集群部署

#### 1. ProxySQL 部署方式一些说明

ProxySQL 很轻量，作为应用的数据库连接代理，最好放置在客户端应用就近点部署。例如可以和应用在同一个 kubernetes 集群中，又或者针对单独应用当作 sidecar 和应用放在同一个 POD 内。

#### 2. ProxySQL 配置文件一些基础

主要是下面 4 块配置

- `mysql_query_rules`

  配置匹配 SQL 语句规则，按照优先级顺序匹配到规则后转发到指定 MySQL 服务器组

  路由概述可以参考：https://www.cnblogs.com/f-ck-need-u/p/9300829.html

- `mysql_servers`

  配置可用的 MySQL 服务器组

- `mysql_users`

  配置可用的 MySQL 用户，需要连接的数据库账号都要添加在这里

- `proxysql_servers`

  配置 ProxySQL 集群中实例信息

官方wiki参考：https://github.com/sysown/proxysql/wiki/ProxySQL-Cluster#preface



#### 使用 Configmap 存放配置文件

ProxySQL 通常通过登陆管理接口使用 SQL 语句热更新来进行配置，使用配置文件只用来完成初始化。在 Kubernetes 中使用 configmap 好处是可以集中修改持久化的配置文件，缺点是不能在 proxysql 中热更新后再 load 配置到 disk 持久化。

**生成并创建 `configmap` 配置文件命令参考**：

```
kubectl create configmap proxysql-cnf --from-file=proxysql.cnf --dry-run -o yaml > proxysql.cnf.yaml
kubectl apply -f proxysql.cnf.yaml
```

**修改配置可以使用**：

```
kubectl edit cm proxysql-cnf
```

**初始化配置文件参考**：

```
# filename：proxysql.cnf
datadir="/var/lib/proxysql"

admin_variables=
{
    # 配置管理账号，即 6032 端口登陆的管理员账号密码。
    # 默认账号 admin 无法远程登陆，不适用于容器环境
    admin_credentials="proxysql-admin:adminpassword;cluster1:clusterpassword1"
    mysql_ifaces="0.0.0.0:6032"
    refresh_interval=2000
    cluster_username="cluster1"
    cluster_password="clusterpassword1"
    cluster_check_interval_ms=200
    cluster_check_status_frequency=100
    cluster_mysql_query_rules_save_to_disk=true
    cluster_mysql_servers_save_to_disk=true
    cluster_mysql_users_save_to_disk=true
    cluster_proxysql_servers_save_to_disk=true
    cluster_mysql_query_rules_diffs_before_sync=3
    cluster_mysql_servers_diffs_before_sync=3
    cluster_mysql_users_diffs_before_sync=3
    cluster_proxysql_servers_diffs_before_sync=3
}

mysql_variables=
{
    threads=4
    max_connections=2048
    default_query_delay=0
    default_query_timeout=36000000
    have_compress=true
    poll_timeout=2000
    interfaces="0.0.0.0:6033;/tmp/proxysql.sock"
    default_schema="information_schema"
    stacksize=1048576
    server_version="5.1.30"
    connect_timeout_server=10000
    monitor_history=60000
    monitor_connect_interval=200000
    monitor_ping_interval=200000
    # 实际发现 mysql-default_charset 配置没有生效
    mysql-default_charset="utf8mb4"  
    mysql-set_query_lock_on_hostgroup=0
    ping_interval_server_msec=10000
    ping_timeout_server=200
    commands_stats=true
    sessions_sort=true
    # 需要高权限账号，近似于 root
    monitor_username="proxysql_monitor"
    monitor_password="proxysql_monitor"
    monitor_galera_healthcheck_interval=2000
    monitor_galera_healthcheck_timeout=800
}

mysql_replication_hostgroups =
(
    {
        writer_hostgroup=10
        reader_hostgroup=20
        offline_hostgroup=9999
        max_writers=1
        writer_is_also_reader=1
        max_transactions_behind=30
        active=1
    }
)

mysql_servers =
(
    { address="172.17.0.12" , port=3306 , hostgroup=10, max_connections=500 },
    # 读库可以分配权重，测试没生效，建议通过管理接口手动配置
    { address="172.17.0.12" , port=3306 , hostgroup=20, max_connections=500, weight=0 },
    { address="172.17.0.130" , port=3306 , hostgroup=20, max_connections=500, weight=70 } 
)

mysql_query_rules =
(
    {
        rule_id=10
        active=1
        match_digest="^SELECT.*FOR UPDATE$"
        destination_hostgroup=10
        apply=1
    },
    {
        rule_id=20
        active=1
        match_digest="^SELECT"
        destination_hostgroup=20
        apply=1
    },
        rule_id=100
        active=1
        match_pattern="^SELECT .* FOR UPDATE"
        destination_hostgroup=10
        apply=1
    },
    {
        rule_id=200
        active=1
        match_pattern="^SELECT .*"
        destination_hostgroup=20
        apply=1
    },
    {
        rule_id=300
        active=1
        match_pattern=".*"
        destination_hostgroup=10
        apply=1
    }
)

mysql_users =
(
    { username = "user1", password = "password1", default_hostgroup = 10, transaction_persistent = 0, active = 1 },
    { username = "user2", password = "password2", default_hostgroup = 10, transaction_persistent = 0, active = 1 }
)

proxysql_servers =
(
    { hostname = "proxysql-0.proxysqlcluster", port = 6032, weight = 1 },
    { hostname = "proxysql-1.proxysqlcluster", port = 6032, weight = 1 }
)
```

#### 部署之前准备

根据实际配置文件中内容修改下面命令或配置文件

1. 由于配置文件中需要指定的 monitor 账号，需要提前在 MySQL 内准备好，并添加`USAGE`权限。

```mysql
GRANT USAGE ON *.* TO 'proxysql_monitor'@'%';
```

2. 配置文件中 `proxysql_servers` 里面的 `hostname` 指定的 `proxysql-0.proxysqlcluster` 需要提前创建无头服务，方便配置文件初始化时直接解析自己

   声明 headless svc 文件配置参考：

```yaml
# filename: proxysql-admin-svc-headless.yml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: proxysql
  name: proxysqlcluster
spec:
  clusterIP: None  # 指明为无头服务，使解析直接返回 POD 的 IP 地址
  ports:
  - name: proxysql-admin
    port: 6032
    protocol: TCP
    targetPort: 6032
  selector:
    app: proxysql  # 绑定接下来创建的服务
  sessionAffinity: None
  type: ClusterIP
```

#### 部署 workload 类型为 StatefulSet 的 Proxysql 集群

StatefulSet 声明文件参考：

```yaml
# filename: proxysql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: proxysql
  labels:
    app: proxysql
spec:
  replicas: 2
  serviceName: proxysqlcluster
  selector:
    matchLabels:
      app: proxysql
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: proxysql
    spec:
      restartPolicy: Always
      containers:
      - image: proxysql/proxysql:2.0.8
        name: proxysql
        volumeMounts:
        - name: proxysql-cnf
          mountPath: /etc/proxysql.cnf
          subPath: proxysql.cnf
        ports:
        - containerPort: 6033
          name: proxysql-mysql
        - containerPort: 6032
          name: proxysql-admin
      volumes:
      - name: proxysql-cnf
        configMap:
          name: proxysql-cnf
---
# filename: proxysql-svc-nodeport.yaml
# 根据需求是否创建 nodeport
apiVersion: v1
kind: Service
metadata:
  annotations:
  labels:
    app: proxysql
  name: proxysql
spec:
  ports:
  - name: proxysql-mysql
    port: 6033
    protocol: TCP
    targetPort: 6033
    nodePort: 6033
  - name: proxysql-admin
    nodePort: 6032
    port: 6032
    protocol: TCP
    targetPort: 6032
  selector:
    app: proxysql
  type: NodePort 
```

apply 上面文件完成部署。

#### 验证启动状态

```bash
kubectl get po,svc
```

### 二、 添加 Prometheus 监控支持

使用 [proxysql-export](https://github.com/percona/proxysql_exporter) 查询 PorySQL 状态暴露指标给 prometheus 。

创建 Deployment，声明文件参考：

```yaml
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: proxysql-exporter
spec:
  replicas: 1
  selector:
    matchLabels:
      name: proxysql-exporter
  template:
    metadata:
      labels:
        name: proxysql-exporter
        app: proxysql-exporter
      annotations:
        prometheus.io/path: /metrics
        prometheus.io/port: "42004"
        prometheus.io/scrape: "true"
    spec:
      containers:
      - name: proxysql-exporter
        image: epurs/proxysql_exporter
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 42004
        env:
        - name: DATA_SOURCE_NAME
          value: "proxysql-admin:adminpassword@tcp(proxysql.default:6032)/"
      # args:
      # - -web.listen-address=:42004
```


### 三、ProxySQL 常见问题

#### 1. 添加用户

需要增加新用户访问时，登陆 6032 管理接口，参考如下命令查看添加：

```mysql
-- 查看当前用户
select * from mysql_users            -- 内存数据库中配置
select * from runtime_mysql_users    -- 当前运行时中生效的配置，同 MySQL 密码为 * 开头的 hash 结果
-- 添加新用户
insert into mysql_users(username,password,default_hostgroup) values('sqlsender','P@ssword1!',10)  -- 此时密码为明文
-- 加载配置
load mysql users to runtime          -- 此时已生效
save mysql users to memory;          -- 可选：将 hash 密码加载到内存数据库中
-- 验证
select * from mysql_users 
```

参考：[mysql_users表中用户的密码管理](https://www.cnblogs.com/f-ck-need-u/p/9286922.html#52-mysql_users%E8%A1%A8%E4%B8%AD%E7%94%A8%E6%88%B7%E7%9A%84%E5%AF%86%E7%A0%81%E7%AE%A1%E7%90%86) 



#### 2. 添加/修改 backend 数据库

需要增加新数据库实例到数据库组时，登陆 6032 管理接口，参考如下命令查看和添加：

```mysql
-- 查看当前后端情况  select * from mysql_servers\G
select hostgroup_id,hostname,port,status,weight from mysql_servers; 
-- 
-- 需要平滑下掉某个节点，可以将状态改为 OFFLINE_SOFT，该后端将不再有新链接建立，但是旧链接不会主动释放。效果和把 weight 设置为 0 相同。
-- 如果需要立即停掉所有链接，可以 DELETE 实例条目，或者修改状态为 OFFLINE_HARD
UPDATE mysql_servers SET status='OFFLINE_SOFT' WHERE hostname='172.17.0.1'
-- 
-- 添加后端 MySQL 节点（简化版本）
insert into mysql_servers(hostgroup_id,hostname,port) 
values(10,'172.17.0.0',3306)
-- 添加后端 MySQL 节点（完整版本）
NSERT INTO mysql_servers(hostgroup_id,hostname,port,status,weight,compression,max_connections) VALUES ('10','172.17.0.0','3306','ONLINE','10','0','500');
-- 
-- 加载配置
load mysql servers to runtime;
-- 验证
select * from mysql_servers\G
```



#### 3. 读写分离配置

比较通用的读写分离规则，`rule_id` 值越小优先级越高，先匹配到会直接按照规则执行 ：

```mysql
INSERT INTO mysql_query_rules (rule_id,active,match_digest,destination_hostgroup,apply)
VALUES
(10,1,'^SELECT.*FOR UPDATE$',10,1),
(20,1,'^SELECT',20,1);
```

改配置官方认为只能作为例子使用，详细创建及优化流程参考官方 wiki 说明，https://github.com/sysown/proxysql/wiki/ProxySQL-Read-Write-Split-(HOWTO)



#### 4. utf8mb4 字符集问题

使用中遇到了配置文件中指定数据库变量 `mysql-default_charset="utf8mb4"` 不生效的问题，需要部署完集群后，执行：

```mysql
SET mysql-default_charset='utf8mb4';
LOAD MYSQL VARIABLES TO RUNTIME;
-- SAVE MYSQL VARIABLES TO DISK;
select * from  global_variables ;
```



#### 5. 查看读写分离路由统计

```mysql
select * from stats_mysql_query_rules;
select hostgroup,schemaname,username,digest_text from stats_mysql_query_digest;
```



#### 6. SQL 查询遇到报错 ProxySQL Error: connection is locked to hostgroup 100 but trying to reach hostgroup 200

修改配置

```mysql
set mysql-set_query_lock_on_hostgroup=0;
```

解释如下：

Possible values for `mysql-set_query_lock_on_hostgroup`:

- 0 : legacy behavior , before 2.0.6
- 1 : (default) . SET statements that cannot be parsed correctly disable
  both multiplexing AND routing. Attempting to route traffic while a
  connection is linked to a specific backend connection will trigger
  an error to be returned to the client

Issue 参考：

https://github.com/sysown/proxysql/pull/2156

https://github.com/sysown/proxysql/issues/2234

#### Reference

部署参考：

https://severalnines.com/database-blog/proxysql-native-clustering-kubernetes

配置详解参考：

https://www.cnblogs.com/f-ck-need-u/p/7586194.html#auto_id_4

github 地址：

https://github.com/sysown/proxysql
