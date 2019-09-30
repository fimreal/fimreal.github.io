---
title: "在 Kubernetes 上挂载使用腾讯云对象存储 COS 的一种配置方法"
date: 2019-09-29T10:05:00Z
lastmod: 2019-09-30T07:30:00Z
draft: false
keywords: ["Kubernetes","COS"]
description: "在 Kubernetes 上挂载使用腾讯云对象存储 COS 的一种配置方法"
tags: ["Kubernetes"]
categories: ["Kubernetes","ops"]
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

目前 Kubernetes 自身网络卷挂载尚不支持共有云提供的对象存储产品，但是可以通过 Kubernetes 自身所拥有的功能利用一些小技巧将对象存储挂载到集群内实现共享读写的目的。

通过搜索，我们可以知道阿里云给自己的 Kubernetes 平台做了挂载 OSS 的插件，最佳实践可以参考[K8S有状态服务-OSS存储使用最佳实践](<https://yq.aliyun.com/articles/640212>)，理论上使用阿里云 OSS 产品都可以自行[安装插件](<https://help.aliyun.com/document_detail/86785.html?spm=a2c4g.11186623.2.10.46bb7db2T54nWw#concept-wrq-dvs-vdb>)来使用，使用阿里云产品的可以优先参考文档说明，或者提交工单寻求帮助。文中所举例使用的是腾讯云的 COS，用法则是参考 AWS 博客写的 k8s 挂载 S3 说明 [Kubernetes pods used shared S3 storage](<https://amazonaws-china.com/cn/blogs/china/use-u3fs-as-shared-storage-to-kubernetes-pod/>) 。
<!--more-->

### 准备

前言中提到的三家对象存储产品，AWS s3、qcloud cos、aliyun oss，在 Linux 中挂载网络存储，都是基于使用开源的[s3fs-fuse](https://github.com/s3fs-fuse/s3fs-fuse) 软件，只是密码文件写法略有不同，这里需要在 github 找到并下载对应的源码包编译安装。

为了挂载使用云服务器产品，需要获取相应的 `AccessToken` ，参考对应云产品文档熟悉裸机挂载办法，文档中写的很清楚所以这里不再敖述。

此处实现办法参考 s3 挂载方法改写为适用于腾讯云 `cosfs` 的配置，通过使用 Kubernetes 的 `daemonset` 模式在需求的节点部署挂载 `cosfs` 的容器镜像，然后通过卷共享模式提供给新创建的业务容器使用。

#### 创建挂载镜像

同样参考 s3 容器创建的文件编写 `Dockerfile`。测试时发现，使用 `alpine:3.10` 编译 `cosfs` 会报异常错误，但是可以正常编译 `s3fs` ，编译成功生成库文件后再编译其他两家的挂载工具则可以实现三合一镜像。

创建 `Dockerfile`，内容如下：

```dockerfile
FROM alpine:3.3
ENV COSFS_VERSION=1.0.14
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories &&\
    apk --update --no-cache add fuse alpine-sdk automake autoconf libxml2-dev fuse-dev curl-dev bash &&\
    cd /tmp &&\
    wget https://github.com/tencentyun/cosfs/archive/v${COSFS_VERSION}.tar.gz &&\
    tar xf v${COSFS_VERSION}.tar.gz &&\
    cd cosfs-${COSFS_VERSION} &&\
    ./autogen.sh &&\
    ./configure --prefix=/usr &&\
    make &&\
    make install &&\
    make clean &&\
    rm -rf /var/cache/apk/* /tmp/* &&\
    apk del automake autoconf
```

创建基础环境镜像

```bash
docker build . -t epurs/cosfs:base
```

测试可以使用我打包好的：`epurs/cosfs:base`

想要直接封装更完善的启动方式可以继续在 `Dockerfile` 中添加内容，比如：

```dockerfile
FROM epurs/cosfs:base
ENTRYPOINT /usr/bin/cosfs -f ${BUCKET_NAME} ${MOUNT_PATH}
```

#### 在 Kubernetes 中创建 DaemonSet 挂载对象存储

这一步如果没有现成 yaml 抄会需要大量时间排错，使用时可以参考下面替换变量部分编写适合自己的配置，这里指定不同命名空间不影响存储卷共享。

创建文件 `ds-cosfs-res.yaml`

```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  namespace: cosfs # 需要提前创建
  labels:
    app: cosfs-res
  name: cosfs-res
spec:
  template:
    metadata:
      labels:
        app: cosfs-res
    spec:
    # tolerations:
    #   - key: node-role.kubernetes.io/master
    #     effect: NoSchedule
    # affinity:
    #   nodeAffinity:
    #     requiredDuringSchedulingIgnoredDuringExecution:
    #       nodeSelectorTerms:
    #       - matchExpressions:
    #         - key: kubernetes.io/hostname
    #           operator: NotIn  # 可选 In 或者 NotIn
    #           values:
    #           - k8s-master-01
    #           - k8s-master-02
      containers:
      - name: cosfs-res
        image: epurs/cosfs:base
        securityContext:
          capabilities:
            add:
            - SYS_ADMIN
          privileged: true
        command: ["/usr/bin/cosfs","-opasswd_file=/etc/cos/passwd-cosfs","${BUCKET_NAME}","${MOUNT_PATH}","-ourl=http://cos.ap-beijing.myqcloud.com","-f"]
        volumeMounts:
        - name: passwd-cosfs
          mountPath: /etc/cos/
        - name: devfuse
          mountPath: /dev/fuse
        - name: cosfs-res
          mountPath: ${MOUNT_PATH}:shared
    # imagePullSecrets:
    # - name: epurs
      volumes:
      - name: passwd-cosfs
        secret:
          secretName: passwd-cosfs
          defaultMode: 0600
      - name: devfuse
        hostPath:
          path: /dev/fuse
      - name: cosfs-res
        hostPath:
          path: ${MOUNT_PATH} # 保证宿主机目录为空否则会警告退出；容器内 mount 节点上是看不到的
```

文件中 `passwd-cosfs`密码配置是使用 secrets 创建文件实现，参考：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: passwd-cosfs
  namespace: cosfs
type: Opaque
data:
  passwd-cosfs: ******
```

`******`代指 base64 后的密码文件内容，例如文件中有两个 bucket 路径时，命令行使用如下命令：

```bash
bash-4.3# base64 <<<"buckername1:AKID123456789:123456789
> buckername2:AKID123456789:123456789"
YnVja2VybmFtZTE6QUtJRDEyMzQ1Njc4OToxMjM0NTY3ODkKYnVja2VybmFtZTI6QUtJRDEyMzQ1
Njc4OToxMjM0NTY3ODkK
```

将生成内容粘贴取代`******`即可。

最后，`kubectl apply -f .` 使两个配置文件生效。

#### 通过共享存储卷测试挂载效果

创建一个临时 pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-cosfs
# namespace: default
spec:
# nodeSelector:
#   kubernetes.io/hostname: k8s-master-03 
  tolerations:
    - key: node-role.kubernetes.io/master
  containers:
  - image: alpine
    command: ["sleep","3600"]
    name: test-cosfs
    volumeMounts:
    - name: cosfs-res
      mountPath: ${MOUNT_PATH}:shared
  volumes:
  - name: cosfs-res
    hostPath:
      path: ${MOUNT_PATH}
```

进入 pod 拉起的容器查看挂载目录中文件是否可以正常读写

```bash
kubectl exec -it test-cosfs ash 
```





