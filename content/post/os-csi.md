---
title: "oss/cos 也能动态创建 pv"
date: 2022-09-22T16:06:45+08:00
lastmod: 2022-09-22T16:06:45+08:00
draft: false
keywords: ["kubernetes","csi"]
description: "编写 csi 插件利用 oss/cos 动态创建 pv"
tags: ["kubernetes","csi"]
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
对象存储是云上用到最方便的存储介质，相信有很多人考虑过使用对象存储来给 k8s 提供`persistent volumes`。网上典型的用例如 yandex 写的 csi 插件： [k8s-csi-s3](https://github.com/yandex-cloud/k8s-csi-s3)，可以直接实现底层使用 s3 的 `StorageClass`。可惜该插件并不适配国内阿里和腾讯两家的对象存储，所以这里提供一个改写过的 csi 插件实现同样方便的使用体验。
<!--more-->
## 参考资料
查看国内阿里云和腾讯云的 github 仓库，也可以找到对应的对象存储使用方法，不过两家都是为自己的 k8s 产品写的插件。由于使用 fuse 挂载的命令都 fork 自 s3fs，且存在挂载进程无法自己恢复的问题，所以两家插件都没有直接提供实现动态存储的功能。
腾讯云：[https://github.com/TencentCloud/kubernetes-csi-tencentcloud/blob/master/docs/README_COSFS.md](https://github.com/TencentCloud/kubernetes-csi-tencentcloud/blob/master/docs/README_COSFS.md)
阿里云：[https://github.com/kubernetes-sigs/alibaba-cloud-csi-driver/blob/master/docs/oss.md](https://github.com/kubernetes-sigs/alibaba-cloud-csi-driver/blob/master/docs/oss.md)

既然有了创建 pv 的事例，再去参考 yandex 写的 csi 插件，自然也能写出来同样的东西。（如果使用 aws s3 同类型对象存储，包含 minio，直接使用 yandex-cloud 原仓库代码即可。）
#### yandex-cloud/k8s-csi-s3 源码简单解析
代码部分目录结构如下
```
.
├── cmd
│   └── s3driver
│       └── main.go
├── pkg
│   ├── driver
│   │   ├── controllerserver.go
│   │   ├── driver.go
│   │   ├── driver_suite_test.go
│   │   ├── driver_test.go
│   │   ├── identityserver.go
│   │   └── nodeserver.go
│   ├── mounter
│   │   ├── geesefs.go
│   │   ├── mounter.go
│   │   ├── rclone.go
│   │   └── s3fs.go
│   └── s3
│       └── client.go
└── Dockerfile
```

- cmd 中为启动入口，可以传入两个参数，csi sock 文件路径和所在的节点名称，使用这两个参数创建 csi 驱动。
- pkg/driver 实现了 csi 驱动，由标准的三个组件 controllerserver、identityserver、nodeserver 组成。其中 controllerserver 用来控制 pv 的生命周期，需要实现创建删除快照控制等，可能会用到第三方 sdk 调用创建删除等接口。identityserver 提供驱动的基础信息，不需要大量配置。nodeserver 更复杂一些，需要完成 pv 在 node 上实际的挂载卸载等的函数。
- pkg/mounter 是从 nodeserver 分离出来的包。用来完成实际挂载操作，像 nfs 或者 localpath 本身已经集成的挂载，就不用自己再实现。对于对象存储，需要用到 cosfs、s3fs 等工具。
- pkg/s3 使用 minio sdk，主要用来给对应 pv 创建初始目录。

[
](https://github.com/yandex-cloud/geesefs)
这里面 sdk 连接代码，以及挂载命令部分是需要改写的。
## 使用自编写的 os-csi
sdk 连接部分可参考云厂商对象存储 api 文档，直接复制粘贴使用，而挂载部分按照原仓库代码直接添加需要的命令即可。总的来说改起来还算容易，依葫芦画瓢修改后 csi 插件源码： [https://github.com/fimreal/os-csi/](https://github.com/fimreal/os-csi/)
其中有些固有问题原仓库没有解决，例如 daemonset 容器重启会导致挂载进程退出不会自己恢复，pvc 策略选择 retain 后不能随 pv 删除历史目录等。稳定性速度也依赖挂载命令可靠性，就之前使用 cosfs 体验来看不能让人完全放心。
所以，目前无法保证生产环境100%可用。
#### 1. 创建 os-csi 驱动服务
参考文件：[https://github.com/fimreal/os-csi/tree/main/deploy/kubernetes](https://github.com/fimreal/os-csi/tree/main/deploy/kubernetes)

创建 secret 配置
```yaml
# kubectl apply -f secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: csi-os-secret
  namespace: kube-system
stringData:
  accessKeyID:
  secretAccessKey:
  # endpoint 写访问地址，例如 <region>.xxx.xxx 不包含 bucket 名字
  endpoint:
  # 挂载命令，根据镜像分辨，可选 cosfs、ossfs，也用来区分不同服务端
  mounter:
```

- endpoint 填写 bucket 地址，例如`http://cos.ap-beijing.myqcloud.com`
- mounter 填写挂载命令，腾讯云使用`cosfs`，阿里云使用`ossfs`

部署 provisioner 和 attacher 容器
```bash
kubectl apply -f https://raw.githubusercontent.com/fimreal/os-csi/main/deploy/kubernetes/csi-provisioner.yaml

kubectl apply -f https://raw.githubusercontent.com/fimreal/os-csi/main/deploy/kubernetes/csi-os.yaml

# 查看是否正常启动
kubectl get po -n kube-system | grep csi
```

创建 storageclass
```bash
# kubectl apply -f storageclass.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: csi-os
provisioner: csi.os
parameters:
  # -d,--debug  set log level to debug 
  options: ""
	# 填写 bucket 名称
  bucket: test-xxxx
	# secret 名称和上面创建的一致
  csi.storage.k8s.io/provisioner-secret-name: csi-os-secret
  csi.storage.k8s.io/provisioner-secret-namespace: kube-system
  csi.storage.k8s.io/controller-publish-secret-name: csi-os-secret
  csi.storage.k8s.io/controller-publish-secret-namespace: kube-system
  csi.storage.k8s.io/node-stage-secret-name: csi-os-secret
  csi.storage.k8s.io/node-stage-secret-namespace: kube-system
  csi.storage.k8s.io/node-publish-secret-name: csi-os-secret
  csi.storage.k8s.io/node-publish-secret-namespace: kube-system
```
#### 2. 测试创建 pv
```bash
kubectl apply -f https://raw.githubusercontent.com/fimreal/os-csi/main/deploy/kubernetes/examples/pvc.yaml
kubectl get pvc csi-os-pvc

kubectl apply -f https://raw.githubusercontent.com/fimreal/os-csi/main/deploy/kubernetes/examples/pod.yaml
kubectl get pv

kubectl get po csi-os-test-nginx
```

#### 3. 问题排查
如果遇到 pvc 创建失败，查看 provisioner 日志：
```bash
kubectl -nkube-system logs --tail 100 -l app=csi-provisioner-os -c csi-os
```
如果遇到 pv 创建失败，查看 daemonset pod 日志：
```bash
kubectl -nkube-system logs --tail 100 -l app=csi-os -c csi-os
```

## 其他
我使用到挂载对象存储的场景是自己跨云的 k3s 集群，要求很简单，随处可以挂载，且不易因为网络原因挂掉影响集群稳定性即可。
写 csi 插件也是为了熟悉 golang 编写 k8s 插件的过程，是一次很不错的实践。如果这能对有需要的人提供一些小帮助那就更好了。

