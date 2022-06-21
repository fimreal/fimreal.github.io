---
title: "CentOS 7 制作 rpm 包离线更新 openssh"
date: 2022-02-28T15:47:14+08:00
lastmod: 2022-02-28T15:47:14+08:00
draft: false
keywords: ["openssh","rpm","CentOS 7"]
description: "CentOS 7 制作 rpm 包离线更新 openssh"
tags: ["rpm"]
categories: ["OS"]
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

最近遇到一台内网服务器被扫描到需要修复 openssh 漏洞，系统装的 CentOS 7 自带 openssh 还在用着 7.6p1 的版本，由于没办法直接连接互联网，选择制作 rpm 安装包升级。

制作包以及升级安装过程比较简单，但过程中遇到的一些小问题觉着有必要记录下来。

<!--more-->

## 基础环境

CentOS 7 自带 OpenSSH 7.6p1 版本目前已知存在以下中高风险漏洞：

- CVE-2020-15778 影响范围：OpenSSH <= 8.3

- CVE-2021-41617 影响范围：OpenSSH 版本6.2 - 8.8 

- CVE-2020-14145 影响范围：OpenSSH 版本5.7 - 8.4 

- CVE-2016-20012 影响范围：OpenSSH <= 8.7

漏洞详情查询：[http://www.cnnvd.org.cn/web/vulnerability/querylist.tag](http://www.cnnvd.org.cn/web/vulnerability/querylist.tag) 

服务器上修改过 ssh 配置文件。

## 备份环境

ssh 建立连接创建 tty 后，只要不退出或者因为超时断开就可以保持会话，更新前多开几个窗口就可以大大提升出现故障恢复的可行性。

此环境配置文件修改过，加上 7.6p1 => 公版 8.8p1 ，会改动 pam 文件，一定要备份后操作。

备份包含三个地方，参考命令如下：

```bash
cp -a /etc/ssh /tmp/ssh.bak-`date +%F`
cp -a /etc/pam.d/sshd /tmp/pam.sshd-`date +%F`
cp -a .ssh /tmp/.ssh.bak-`date +%F`
```



## 制作 rpm 包

新购买临时的 CentOS 7.6 云服务器，安装编译工具和 rpm 依赖：

```bash
yum install rpm-build zlib-devel openssl-devel gcc perl-devel pam-devel xmkmf libXt-devel gtk2-devel make libXt-devel imake gtk2-devel
```

下载源码包

```bash
wget https://cdn.openbsd.org/pub/OpenBSD/OpenSSH/portable/openssh-8.8p1.tar.gz
wget https://src.fedoraproject.org/repo/pkgs/openssh/x11-ssh-askpass-1.2.4.1.tar.gz/8f2e41f3f7eaa8543a2440454637f3c3/x11-ssh-askpass-1.2.4.1.tar.gz
```

创建 rpmbuild 目录，拷贝源码包

```bash
mkdir -p rpmbuild/SPECS/ rpmbuild/SOURCES/
cp x11-ssh-askpass-1.2.4.1.tar.gz openssh-8.8p1.tar.gz rpmbuild/SOURCES/
```

解压打包配置

```bash
tar xf openssh-8.8p1.tar.gz openssh-8.8p1/contrib/redhat/openssh.spec -C rpmbuild/SPECS/
```

这时候开始编译会报错，提示库版本不对。

```bash
# rpmbuild -ba openssh.spec
error: Failed build dependencies:
	openssl-devel < 1.1 is needed by openssh-8.8p1-1.el7.x86_64
```

这里修改配置 `rpmbuild/SPECS/openssh.spec` 忽略，在 `BuildRequires: openssl-devel < 1.1`  前添加 "#" 注释后，继续编译

```bash
cd rpmbuild/SPECS/ && rpmbuild -ba openssh.spec
```

编译一长串后会生成文件在 `rpmbuild/RPMS/x86_64/` 下。

```bash
# ll -h
total 4.8M
-rw-r--r-- 1 root root 686K Feb 24 16:41 openssh-8.8p1-1.el7.x86_64.rpm
-rw-r--r-- 1 root root  44K Feb 24 16:41 openssh-askpass-8.8p1-1.el7.x86_64.rpm
-rw-r--r-- 1 root root  25K Feb 24 16:41 openssh-askpass-gnome-8.8p1-1.el7.x86_64.rpm
-rw-r--r-- 1 root root 574K Feb 24 16:41 openssh-clients-8.8p1-1.el7.x86_64.rpm
-rw-r--r-- 1 root root 3.1M Feb 24 16:41 openssh-debuginfo-8.8p1-1.el7.x86_64.rpm
-rw-r--r-- 1 root root 451K Feb 24 16:41 openssh-server-8.8p1-1.el7.x86_64.rpm
```

只需要拷贝升级需要的包，安装

```bash
yum localinstall *.rpm
```

**最后验证配置文件是否被修改，确认监听端口，是否限制 root 登陆等，没问题后重启 sshd**



## 更新 openssh 后遇到的问题



#### 1. 远程登陆不上，sshd 日志显示`no matching host key type found. Their offer: ssh-rsa,ssh-dss [preauth]` 

服务端配置支持客户端验证的密钥格式就好了，通常旧的版本使用 rsa 格式。在 `/etc/ssh/sshd_config` 中添加：

```
UsePAM yes
PubkeyAuthentication yes
PubkeyAcceptedKeyTypes=+ssh-rsa
PubkeyAcceptedAlgorithms +ssh-rsa
HostKeyAlgorithms +ssh-rsa
```

参考：[https://bbs.archlinux.org/viewtopic.php?id=270005](https://bbs.archlinux.org/viewtopic.php?id=270005)

如果只是作为客户端报错，可以给 ssh 命令添加参数`-oHostKeyAlgorithms=+ssh-dss`，或者在配置中添加

```
Host xxx
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedKeyTypes +ssh-rsa
```

参考：[https://askubuntu.com/questions/836048/ssh-returns-no-matching-host-key-type-found-their-offer-ssh-dss](https://askubuntu.com/questions/836048/ssh-returns-no-matching-host-key-type-found-their-offer-ssh-dss)

附： fcos 中 redhat 的证书加密配置

```
Ciphers aes256-gcm@openssh.com,chacha20-poly1305@openssh.com,aes256-ctr,aes128-gcm@openssh.com,aes128-ctr
MACs hmac-sha2-256-etm@openssh.com,hmac-sha1-etm@openssh.com,umac-128-etm@openssh.com,hmac-sha2-512-etm@openssh.com,hmac-sha2-256,hmac-sha1,umac-128@openssh.com,hmac-sha2-512
GSSAPIKexAlgorithms gss-curve25519-sha256-,gss-nistp256-sha256-,gss-group14-sha256-,gss-group16-sha512-
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,ecdh-sha2-nistp256,ecdh-sha2-nistp384,ecdh-sha2-nistp521,diffie-hellman-group-exchange-sha256,diffie-hellman-group14-sha256,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512
HostKeyAlgorithms ecdsa-sha2-nistp256,ecdsa-sha2-nistp256-cert-v01@openssh.com,sk-ecdsa-sha2-nistp256@openssh.com,sk-ecdsa-sha2-nistp256-cert-v01@openssh.com,ecdsa-sha2-nistp384,ecdsa-sha2-nistp384-cert-v01@openssh.com,ecdsa-sha2-nistp521,ecdsa-sha2-nistp521-cert-v01@openssh.com,ssh-ed25519,ssh-ed25519-cert-v01@openssh.com,sk-ssh-ed25519@openssh.com,sk-ssh-ed25519-cert-v01@openssh.com,rsa-sha2-256,rsa-sha2-256-cert-v01@openssh.com,rsa-sha2-512,rsa-sha2-512-cert-v01@openssh.com
PubkeyAcceptedKeyTypes ecdsa-sha2-nistp256,ecdsa-sha2-nistp256-cert-v01@openssh.com,sk-ecdsa-sha2-nistp256@openssh.com,sk-ecdsa-sha2-nistp256-cert-v01@openssh.com,ecdsa-sha2-nistp384,ecdsa-sha2-nistp384-cert-v01@openssh.com,ecdsa-sha2-nistp521,ecdsa-sha2-nistp521-cert-v01@openssh.com,ssh-ed25519,ssh-ed25519-cert-v01@openssh.com,sk-ssh-ed25519@openssh.com,sk-ssh-ed25519-cert-v01@openssh.com,rsa-sha2-256,rsa-sha2-256-cert-v01@openssh.com,rsa-sha2-512,rsa-sha2-512-cert-v01@openssh.com
CASignatureAlgorithms ecdsa-sha2-nistp256,sk-ecdsa-sha2-nistp256@openssh.com,ecdsa-sha2-nistp384,ecdsa-sha2-nistp521,ssh-ed25519,sk-ssh-ed25519@openssh.com,rsa-sha2-256,rsa-sha2-512
```



#### 2. 远程登陆报错，sshd 日志显示 `PAM unable to dlopen(/usr/lib64/security/pam_stack.so): /usr/lib64/security/pam_stack.so: cannot open shared object file` 

pam 配置问题，8.8p1 版本 openssh 把 pam 文件更新了，可以用之前备份的还原。或者使用原始 CentOS 的配置覆盖，参考：

```
# cat /etc/pam.d/sshd
#%PAM-1.0
auth       substack     password-auth
auth       include      postlogin
account    required     pam_sepermit.so
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    optional     pam_motd.so
session    include      password-auth
session    include      postlogin
```



#### 3. `/etc/ssh/` 下出现`*.rpmnew`文件

安装 rpm 包时会有报警，没有自动覆盖掉原来的配置文件，网上查了下可以不做修改。

可以把 moduli 文件替换成新的。谨慎点的话，可以把原有配置同步对比下，删掉 rpmnew 文件。

