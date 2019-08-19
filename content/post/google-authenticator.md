---
title: "使用 google_authenticator 生成动态密码加固 ssh 登陆"
date: 2019-06-06T02:40:00
lastmod: 2019-05-06T10:12:00
draft: false
keywords: ["Two Factors Authentication","sshd","secure"]
description: "使用 google_authenticator 生成动态密码加固 ssh 登陆"
tags: ["sshd","secure",]
categories: ["Linux"]
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
利用 google 动态口令生成密码给 sshd 做两步验证实现安全加密。
又或者甩掉固定的密码，来使用手机 app 生成的动态密码登陆服务器。
这只是一些 `pam` 配置实现的小技巧。
<!--more-->
### 前言
早起看到群里发公众号推运维小知识， 介绍用 Google Authenticator（谷歌身份验证器）这个工具来加固 sshd 验证。这个加固办法其实出现的时间比较早了，目的是为了解决中间人攻击问题，谷歌身份验证器是一个可以基于时间来生成密码用于验证的手机端/网页端 app，可以避免同一密码被多次使用。

这么有趣的功能当然要好好把玩一下。但是要注意，文中出现所有命令都不保证**幂等**性，建议读懂意思后通过编辑器修改。

### 准备环境
这里使用 CoreOS 服务器，来测试安装体验。

测试计划主要用到两种 Liunx 环境，`Ubuntu(debian)` 和 `Fedore(rhel/centos)` ，这里 `Fedore` 可以视作 `CentOS` ，全文除了安装，配置上各大型版几乎没有区别。其中 `Ubuntu` 使用 Docker 镜像创建， `Fedore` 使用 CoreOS 自带的 `toolbox` 环境，用来测试。

### 安装 google-authenticator

这里安装只做最简配置，如果 `Fedore` 想要使用二维码手机扫描，则需要安装 `qrencode`，默认 `ubuntu` 会自己装上。

**Fedore**

使用 dnf 安装 google-authenticator 即可。
dnf 是 yum 升级版，向后兼容 yum 子命令，Fedore 中 yum 被软链接指向 dnf-3，在 CentOS7 中配置相同。

```bash
dnf install google-authenticator openssh-server -y
```

**Ubuntu**

默认容器启动 ubuntu-18.04 版本，debian 系安装包都是一致的。

换用支持 `http` 的中科大源，安装：

```bash
sed -i.bak 's/archive.ubuntu.com/mirrors.ustc.edu.cn/g' /etc/apt/sources.list
apt update && apt install libpam-google-authenticator openssh-server -y
```

**Alpine**

Alpine Linux 的 apk 源中也有包，略麻烦一点，安装办法可以参考 [wiki](https://wiki.alpinelinux.org/wiki/Two_Factors_Authentication_With_OpenSSH)

```bash
apk add google-authenticator openssh-server-pam
```

**Arch/Manjaro**

应该不用多说，可以参考 [wiki](https://wiki.archlinux.org/index.php/Google_Authenticator_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)


### 初始化配置

在 `Ubuntu` 容器内执行初始化，与 `Fedore` 初始化配置过程一样：

```bash
root@2a52e5dc9569:/# google-authenticator

Do you want authentication tokens to be time-based (y/n) y
 <这里是自动生成的二维码及链接，可以扫码绑定手机 app >
Your new secret key is: 2MX4BZPDXZONH6SVUPMUC3SO5M    (用于绑定手机 app 的密钥)
Your verification code is 768806                      (验证码)
Your emergency scratch codes are:                     (备用令牌码，用一个少一个)
  43344880
  98680948
  43923070
  26622686
  60782611

Do you want me to update your "/root/.google_authenticator" file? (y/n) y
(是否覆盖 "/root/.google_authenticator" 文件，如果之前生成过文件需要谨慎确认)

Do you want to disallow multiple uses of the same authentication
token? This restricts you to one login about every 30s, but it increases
your chances to notice or even prevent man-in-the-middle attacks (y/n) y
(是否拒绝多次使用使用相同的令牌？这将限制你每30s仅能登录一次，但会提醒/阻止中间人攻击。)

By default, a new token is generated every 30 seconds by the mobile app.
In order to compensate for possible time-skew between the client and the server,
we allow an extra token before and after the current time. This allows for a
time skew of up to 30 seconds between authentication server and client. If you
experience problems with poor time synchronization, you can increase the window
from its default size of 3 permitted codes (one previous code, the current
code, the next code) to 17 permitted codes (the 8 previous codes, the current
code, and the 8 next codes). This will permit for a time skew of up to 4 minutes
between client and server.
Do you want to do so? (y/n) n
(是否将窗口时间由1分30秒增加到约4分钟？这将缓解时间同步问题。)

If the computer that you are logging into isn't hardened against brute-force
login attempts, you can enable rate-limiting for the authentication module.
By default, this limits attackers to no more than 3 login attempts every 30s.
Do you want to enable rate-limiting? (y/n) y
(是否启用此模块的登录频率限制，登录者将会被限制为最多在30秒内登录3次。)
```

上面这步会在 `/root/.google_authenticator` 生成验证文件。

然后再打开 `sshd` 配置中 `pam` 模块，并启用交互式登陆，其实就是打开 `ChallengeResponseAuthentication` 和 `UsePAM` 选项。

```bash
sed -i.bak '1i UsePAM yes\nChallengeResponseAuthentication yes' /etc/ssh/sshd_config
```

添加新的 pam 支持，除了 `Alpine Linux` 默认都会有 `/etc/pam.d/sshd` 配置文件，这里直接把 google_authenticator 放在第一行加载即可。

```bash
sed -i.bak '1i auth            sufficient      pam_google_authenticator.so' /etc/pam.d/sshd
```

补充说明：`sufficient` 配置，指明可以使用 `google_authenticator` 进行验证登陆，验证成功后，如果需要配置两步验证，把 `sufficient` 改为 `required` 即可。

pam 控制命令简单说明：

- **required** 必须要验证的模块。在所有步骤验证完毕后返回验证信息。
- **requisite** 必须要验证的模块。出错立刻返回错误信息。
- **sufficient** 验证成功即通过的模块。成功后不会立即返回成功信息。
- **optional** 失败时会忽略的模块。
- **include** 验证时加载其他配置文件。
- **substack**

### 手机端配置

这里不考虑安装 `qrencode` 的情况，差别只在于绑定手机 app 的方法，一个是手动输入，一个是扫码而已。

手机端下载安装： IOS 在 Appstore 搜索 "Authenticator"；Android 搜索 "谷歌动态口令" 。

安装打开后，点击右上角 “+” ，选择 扫描二维码或者输入验证码

![](https://od.epurs.com/images/2019/06/06/OrQR5XLo5p/google_authentication_0.png)

在系统中可以查看 `/root/.google_authenticator` 文件找到配置的验证码

手机端输入后保存

![](https://od.epurs.com/images/2019/06/06/yKcQVHE4DR/google_authentication_1.png)


### 验证测试

上述配置完成后，手动打开 `sshd` 服务，验证验证连接过程是否符合我们的需求。

由于容器环境新安装的 `sshd` 服务还没有自动初始化配置，需要额外执行一些步骤。

```bash
echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config  ## 允许 root 用户登陆
ssh-keygen -A       # 生成服务器 key
mkdir /run/sshd     # 创建运行环境目录
hostname -I         # 拿到容器 IP
/usr/sbin/sshd -d   # 前台运行 sshd 服务
```

测试连接：

```bash
ssh -v 172.17.0.1
... ...              (公钥验证)
ECDSA key fingerprint is SHA256:/9Z8rQasLV5qsSVfwMc+vsvanpNpoWYd2B2c31Uc6lk.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '172.17.0.6' (ECDSA) to the list of known hosts.
debug1: rekey after 134217728 blocks
debug1: SSH2_MSG_NEWKEYS sent
debug1: expecting SSH2_MSG_NEWKEYS
debug1: SSH2_MSG_NEWKEYS received
debug1: rekey after 134217728 blocks
debug1: SSH2_MSG_EXT_INFO received
debug1: kex_input_ext_info: server-sig-algs=<ssh-ed25519,ssh-rsa,rsa-sha2-256,rsa-sha2-512,ssh-dss,ecdsa-sha2-nistp256,ecdsa-sha2-nistp384,ecdsa-sha2-nistp521>
debug1: SSH2_MSG_SERVICE_ACCEPT received
debug1: Authentications that can continue: publickey,password,keyboard-interactive
debug1: Next authentication method: publickey
debug1: Trying private key: /root/.ssh/id_rsa
debug1: Trying private key: /root/.ssh/id_dsa
debug1: Trying private key: /root/.ssh/id_ecdsa
debug1: Trying private key: /root/.ssh/id_ed25519
debug1: Trying private key: /root/.ssh/id_xmss
debug1: Next authentication method: keyboard-interactive
Verification code: <输入手机上动态密码>
Password: <系统内对应用户密码>
```

如果配置和前面一样，这里输入手机显示的正确动态密码后，可以直接登陆服务器。

如果输入错误，会进一步验证登陆用户对应的密码，输入正确密码，也可登陆。


### 扩展配置

#### 1. 将 `Ubuntu` 中的配置应用到 `Fedore` (其他)系统中

目的：不同发行版，或者说不同主机配置迁移，最终共享同一个配置，使手机端动态密码可以登陆不同主机。

在先前配置中已经提到过 `Fedore` 系统安装过程，这步进行完成后，先跳过 `google-authenticator` 初始化步骤。

保证下面配置正确：

```bash
sed -i.bak '1i UsePAM yes\nChallengeResponseAuthentication yes' /etc/ssh/sshd_config               # 确保打开 pam 支持 和 交互式验证
sed -i.bak '1i auth            sufficient      pam_google_authenticator.so' /etc/pam.d/sshd  # pam 中添加 google_authenticator 验证模块
```

将配置文件复制到 `~/.google_authenticator`，或者通过编辑器编辑添加该文件。文件要确保属主正确，权限为 `600` 或者更低。如果是 root 用户，需要确认 sshd 配置可以远程登陆。

配置完成重启 sshd 服务即可。


#### 2. 禁用账号密码登陆

目的：当使用密钥登陆时，不想同时开启密码登陆。

注意： 该配置需要 openssh 6.2 以上支持，参考 [Arch wiki](https://wiki.archlinux.org/index.php/OpenSSH#Two-factor_authentication_and_public_keys) 说明。
这里修改 `/etc/ssh/sshd_config` 文件中，`PasswordAuthentication yes` 改为 `PasswordAuthentication no` 不会生效，因为启用 PAM 验证后，pam 配置中默认有基于密码验证。

在上述配置基础上：

`Ubuntu` 系统注释掉 `/etc/pam.d/sshd` 中 `@include common-auth` 即可关闭密码验证。
`Fedore` 系统注释掉 `/etc/pam.d/sshd` 中 `auth       substack     password-auth` 即可关闭密码验证。

同理，不开启密钥登陆，这时将只能通过动态密码登陆。别忘了初始化时还有几个一次性备用密码。

#### 3. 使用浏览器插件查看动态密码

Chrome 插件：https://t.cn/E9oafuV

Firefox 插件：https://t.cn/zjaEAlS
