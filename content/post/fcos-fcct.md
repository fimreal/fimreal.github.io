---
title: "从 CoreOS 迁移到 Fedora CoreOS 之 用 fcct 初始化 fcos"
date: 2021-09-10T09:49:22+08:00
lastmod: 2021-09-10T10:02:42+08:00
draft: false
keywords: ["CoreOS","fedora","fcos"]
description: ""
tags: ["CoreOS","fedora","fcos"]
categories: ["CoreOS","fedora","fcos"]
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

安装 Fedora CoreOS 系统需要用到 ignition 配置，初看起来很复杂，其实做起来很简单。简单记录创建 fcc 配置文件，并在 vmware 安装 fcos ，转化 qcow2 镜像的过程。

<!--more-->

## 0. 什么是点火配置 Ignition configurations

​		Ignition configurations 可以理解为 cloud-init 同类型的初始化配置，用户可以按照指定语法创建 yaml 文件，使用 `fcct`命令行工具生成最终可用的 JSON 格式文件，供 Fedora CoreOS 安装时读取从而达成初始化配置。



​		由于  Fedora CoreOS 使用 ostree 管理文件结构，根分区等默认为只读挂载，如果需要自定义目录结构（其实还是通过软链接到 `/var/` 内实现），就必须用到点火文件来配置。



官方说明文档：

https://docs.fedoraproject.org/en-US/fedora-coreos/producing-ign/



## 1. 创建一个简单的点火配置 

按照官方文档中描述，简单的点火配置 demo  ：

```yaml
variant: fcos
version: 1.4.0
passwd:
  users:
    - name: core
      ssh_authorized_keys:
        - ssh-rsa AAAA...
```

默认 fcos 中已经内置了 core 用户，上面 yaml 配置在安装系统时同时配置了公钥，可以据此添加自定义用户和相应密钥。

习惯使用密码的可以参考下面写法，创建一个用户，加入 docker 组，拥有 sudo 权限，并添加密码登陆方式。

其中密码加密方式使用命令： `mkpasswd --method=yescrypt`。

```yaml
variant: fcos
version: 1.1.0
passwd:
  users:
    - name: xiaoming
      groups:
        - docker
        - sudo
      password_hash: $y$j9T$avmZo2f7txYwPs3H02yTk/$3sXp/LMxj7.lTTGadKyu5qb8PJzdXhM44zYQIr3AyV4
      home_dir: /home/xiaoming
      shell: /bin/bash
storage:
  files:
    - path: /etc/ssh/sshd_config.d/99-AllowPasswordLogin.conf
      mode: 0644
      contents:
        inline: |
          PasswordAuthentication yes
```



## 2. 将点火配置转换为 .ign 配置文件

上面创建的 yaml 文件没法直接使用，需要用 butane 工具生成 JSON 格式 .ign 文件。

fcct [ butane ] 很多发行版仓库中没有预装，安装可以参考：

https://docs.fedoraproject.org/en-US/fedora-coreos/producing-ign/

或者直接使用官方给的容器镜像来帮助生成：

```bash
podman run -i --rm quay.io/coreos/butane:release \
       --pretty --strict < your_config.yaml > transpiled_config.ign
```

这里给出一个网页在线转换工具：

https://fcct.epurs.com/

左侧输入 yaml 文件，点击 Transpile 即可生成 json 格式文件。再点击 Generate URL From FCC 即可生成 ign 文件的 url，可以直接用于系统安装。



## 3. 开始安装 

安装可以使用`coreos-installer` 在线下载镜像，考虑到国内网络因素，推荐下载离线镜像。

这里使用 Vmware 通过 ISO 方式安装，裸机安装大同小异。

ISO 文件下载地址：

https://getfedora.org/en/coreos/download?tab=metal_virtualized&stream=stable&arch=x86_64

https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/34.20210919.3.0/x86_64/fedora-coreos-34.20210919.3.0-live.x86_64.iso



虚拟机启动后进入 LiveCD 模式，成功加载登陆后会给出安装命令提示。

```bash
sudo coreos-installer install /dev/sda \
    --ignition-url https://example.com/example.ign
```

`lsblk` 先确认需要安装的磁盘分区，数据无价。



确认无误后，用自己的 ign 地址安装系统：

```bash
sudo coreos-installer install /dev/sda --ignition-url https://raw.githubusercontent.com/fimreal/fcc/main/fcc.ign
```



后面不需要再人工干预，安装后会提示重启系统。



## 4. 我的点火配置文件

https://github.com/fimreal/fcc/blob/main/ignition.yaml



配置修改和增加可以参考官方文档中 System Configuration 部分：

https://docs.fedoraproject.org/en-US/fedora-coreos/



## 5. 补充：vmware 生成 qcow2 镜像

安装不算麻烦，后续想要将镜像导入到云服务器还需要些许努力。

首先免费版本的 VMware Fusion 不支持导出镜像，需要在网上找个序列号升级到专业版本解锁额外功能。~~太坑了！~~

前面安装的 CoreOS 默认就是 DHCP 配置网络，不用安装 cloud-init 等工具就可以在云平台跑起来，缺少的控制台重置密码功能麻烦点也能自己改，所以不需要再做额外配置。

将装好的虚拟机导出为 OVF 格式，导出速度很快，OVF 其实就是配置文件，关键是要单独打包出来的 vmdk 磁盘镜像文件。

获取到 vmdk 文件后，使用 `qemu-img` 转换格式：

```bash
qemu-img convert -f vmdk -O qcow2 Fedora-disk1.vmdk Fedora-disk1.qcow2
```

转换完成可以查看镜像实际大小：

```bash
qemu-img info Fedora-disk1.qcow2
```

后续根据云厂商说明文档导入进去，开机 ssh 即可。

