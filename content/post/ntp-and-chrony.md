---
title: "Linux 时间同步服务 -- ntp 和 chrony"
date: 2018-01-10T17:05:13Z
lastmod: 2019-04-12T11:50:57Z
draft: false
keywords: ["Linux","ntp","chrony"]
description: "Linux 时间同步服务 ntp 和 chrony 说明"
tags: ["Linux","时间同步"]
categories: ["Linux","ops"]
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
Linux 时间同步工具，ntp 和 chrony 详细介绍，简单对比。
<!--more-->

## NTP 时间协议
#### NTP（网络时间协议）是什么？
NTP（Network Time Protocol，网络时间协议）是用来使网络中的各个计算机时间同步的一种协议。它的用途是把计算机的时钟同步到世界协调时UTC(Universal Time Coordinated,世界协调时），UTC 是由原子钟报时的国际标准时间，而 NTP 获得 UTC 的时间来源可以是原子钟、天文台、卫星，也可从 Internet 上面获取。其精度在局域网内可达0.1ms，在互联网上绝大多数的地方其精度可以达到1-50ms。
补充知识：抢票软件等就是提前发送已知的数据包，时间一到服务器开始接受就会第一时间抢到。
NTP 服务器就是利用 NTP 协议提供时间同步服务的。

在NTP协议中，定义了时间按照服务器等级传播，依据离外部UTC源的远近，将所有服务器归入不同的stratum（层）中，直接从时间源如GPRS(Global Positioning System，全球定位系统）获得时间的服务器称之为stratum-1，而后依次序递归传播给下层服务器stratum-2、stratum-3…，层的总数限制在15以内。

#### 为什么要使用 ntpd 而不是 ntpdate ？

首先要理解，ntpdate 和 ntpd 是互斥的，两者不能同时使用。而推荐使用 ntpd 的原因就是，ntpd 是通过步进式平滑的逐渐调整时间，而 ntpdate 则是断点式更新时间。具体来说，当服务器时间落后于标准时间，ntpd 服务会在一段时间内逐步微调校准到标准时间，而 ntpdate 则是立刻把时间调整到标准时间，如果这时候数据库正在写入内容，在其对时间有严格要求的生产环境下，产生的后果是非常严重的。

**注意**：NTP 也存在一定局限性，如果系统时间比正确的时间要快的话，NTP 是不会帮你做调整的，而且当你的时间设置和正确的时间相差很大的时候，NTP 会花上很长一段时间进行同步调整。此外，**当本地时间与标准时间相差30分钟以上时， ntpd 会停止工作**。

这里有一篇文章，结尾很幽默的说明了自己对ntpd和ntpdate同步时间的看法，有兴趣可以查看：
https://www.cnblogs.com/createyuan/p/4301491.html

#### 为什么说ntpd服务器是不安全的？

##### NTP通信协议原理
1. 首先主机启动 NTPd 服务
2. 客户端向 NTP 服务器发送调整时间的信息
3. NTP 服务器接收到请求会发送当前标准时间给客户端
4. 客户端收到来自 NTP 服务器的时间信息后，调整自己系统时钟，实现时间校准。

NTP 服务器为了支持服务需要启用 UDP 123端口。当我们要利用 Tim Server 来进行时间的同步更新时，就需要使用 NTP 软件提供的 `ntpdate` 来连接端口123。说到这里应该明白了，服务需要以 daemon 模式运行持续开放一个端口，存在风险。

## Linux中的时间

#### 时区

现实生活中时间以时区的方式定义，地球绕太阳旋转的24小时中，世界各地的不同时间由 UTC+地区所属时区决定，全球划分为24个不同的时区。比如中国标准时间晚上8点半，可以有以下两种表示方式：

20:00 CST(Chinese Standard Time,中国标准时间)
12:00 UTC(Universal Time Coordinated,世界协调时)

补充：`zdump`命令可以查询对应时区的当前时间

#### Linux中配置时区

Linux 中的 glibc 已提供许多编译好的时区文件，存放于 `/usr/share/zoneinfo` 中，包含大多数国家和城市的时区

![zoneinfo](https://od.epurs.com/api/raw/?path=/images/2018/01/10/timezone.png)

方法一：修改 `/etc/localtime` 文件，该文件定义了当前系统所在的 local time zone，将 `/usr/share/zoneinfo`中的 time zone 文件符号链接至该文件即可。

方法二：使用 `tzselect` 交互式命令修改 `TZ` 变量的值，**注意**这种方法所做更改会覆盖 `localtime` 中的时区设定，如果要使更改长期有效，可以将 `TZ` （例如：Asia/Shanghai）变量的设置写入到 `/etc/profile` 中。

### Linux中时间设置

注意了，在 Linux 系统中存在两个时钟时间，分别是硬件时钟 RTC（Real Time Clock）和 系统时钟（System Clock）。硬件时钟是指的在主板上的时钟设备，也就是通常可以在 BIOS 画面设置的时钟，即使关机状态也可以计算时间；而系统时钟则是指 Kernel 中的时钟，其值是由1970年1月1日00:00:00 UTC时间至当前时间所经历的秒数总和。当 Linux 启动的时候，系统时钟会读取硬件时钟的设定，之后系统时钟独立运作。长时间运行两者可能将会产生误差。另外所有的 Linux 相关指令都是读取系统时钟指定的，如 `date`。

\#补充：系统启动时，内核通过读取RTC来初始化内核时钟,又叫墙上时间，该时间放在xtime变量中。如前所述，Linux内核与RTC进行互操作的时机只有两个：
1) 内核在启动时从RTC中读取启动时的时间与日期；
2) 内核在需要时将时间与日期回写到RTC中。

对于系统时钟查看设置命令date我们都很熟悉了，而对于硬件时钟需要使用额外的命令：

```
hwclock –r, --show        显示硬件时钟与日期
hwclock –s                将系统时钟调整为与目前的硬件时钟一致。
hwclock –w                将硬件时钟调整为与目前的系统时钟一致。
hwclock -s, –set –date="mm/dd/yy hh:mm:ss"  修改硬件时钟时间
```

当两者存在误差时，我们可以利用命令进行调整同步两者时间

```
hwclock -c, --systohc    将系统时间设置为硬件时间（即以date显示系统时间为准）
hwclock -s, --hctosys    将硬件时间设置为系统时钟（即以BIOS显示硬件时间为准）。
```



## ntp软件

ntp 软件（ntp协议）是 CentOS6 自带，CentOS7 默认没有，需要安装；
chrony 软件（同样支持ntp协议），为 CentOS7.2 之后安装自带。

主要介绍这两款 RedHat 系统自带的软件。

额外的：对于调整时区可以用 `tzdata` 软件（Time Zone data），提供各时区对应的显示格式。

**此处用到系统环境**

```bash
$ cat /etc/redhat-release
CentOS Linux release 7.4.1708 (Core)
$ uname -r
3.10.0-693.el7.x86_64

$ getenforce
Disabled
$ systemctl status firewalld.service
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
```

## **ntpd时钟同步服务**

默认 CentOS7 没有安装，可以直接使用yum进行安装配置

```bash
yum install ntp
```

### 配置ntp软件

设置时区：

```bash
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

配置 ntpd 服务主配置文件 `/etc/ntp.conf` ，以下以 CentOS7 中 ntp-4.2.6 为例

```bash
$ tee </etc/ntp.conf /etc/ntp.conf.bak
# For more information about this file, see the man pages
# ntp.conf(5), ntp_acc(5), ntp_auth(5), ntp_clock(5), ntp_misc(5), ntp_mon(5).

driftfile /var/lib/ntp/drift
# 计算本ntp server 与上层ntpserver的频率误差

# Permit time synchronization with our time source, but do not
# permit the source to query or modify the service on this system.
#restrict default nomodify notrap nopeer noquery
restrict default nomodify

# restrict可以限制客户端权限，可以使用的 parameter说明：
# kod            kod技术可以阻止“Kiss of Death “包对服务器的破坏
# nomodity       client可通过ntp进行时间同步，但不能通过 ntpq, ntpc 等更改server参数
# notrap         不提供trap远程登陆 (remote event logging) 功能
# nopeer         不与其它同一层的ntp server进行时间同步
# noquery        客户端不能够使用 ntpq, ntpc 等指令来查询时间服务器，即拒绝ntp时间同步；
# notrust        拒绝无认证的client
# ignore         拒绝所有连接到ntp server的请求
##如果出现问题，可能是ipv6配置出错，可以尝试添加restrict -6 default kod nomodify notrap nopeer noquery  #针对ipv6设置
##或者修改默认为restrict -4 default nomodify notrap nopeer noquery

# Permit all access over the loopback interface.  This could
# be tightened as well, but to do so would effect some of
# the administrative functions.
restrict 127.0.0.1  ##打开允许本地所有操作
restrict ::1
# Hosts on local network are less restricted.
#restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap

# restrict 用来分配指定网段权限，格式如下：
# restrict ［授权同步的网段］ mask ［netmask］ ［parameter］
# 例：restrict 172.16.1.0 mask 255.255.252.0 nomodify
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (https://www.pool.ntp.org/join.html).
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst
server time1.aliyun.com
server time2.aliyun.com
server time3.aliyun.com
server time4.aliyun.com
server time5.aliyun.com prefer
#server 用来设置上一级ntp服务器，这里使用阿里云提供的ntp服务器，parameter说明:
# prefer     最高优先级
# burst      当一个运程NTP服务器可用时，向它发送一系列的并发包进行检测。
# iburst     当一个运程NTP服务器不可用时，向它发送一系列的并发包进行检测。

# 如果无法与上层ntp server通信以本地时间为标准时间
server 127.127.1.0 # local clock
fudge 127.127.1.0 stratum 10
#broadcast 192.168.1.255 autokey	# broadcast server
#broadcastclient			# broadcast client
#broadcast 224.0.1.1 autokey		# multicast server
#multicastclient 224.0.1.1		# multicast client
#manycastserver 239.255.254.254		# manycast server
#manycastclient 239.255.254.254 autokey # manycast client

# Enable public key cryptography.
#crypto

includefile /etc/ntp/crypto/pw

# Key file containing the keys and key identifiers used when operating
# with symmetric key cryptography.
keys /etc/ntp/keys

# Specify the key identifiers which are trusted.
#trustedkey 4 8 42

# Specify the key identifier to use with the ntpdc utility.
#requestkey 8

# Specify the key identifier to use with the ntpq utility.
#controlkey 8

# Enable writing of statistics records.
#statistics clockstats cryptostats loopstats peerstats

# Disable the monitoring facility to prevent amplification attacks using ntpdc
# monlist command when default restrict does not include the noquery flag. See
# CVE-2013-5211 for more details.
# Note: Monitoring will not be disabled with the limited restriction flag.
disable monitor
```

觉着太长不看？简单来说，只要修改server 中上级ntp服务器为离自己较近的服务器即可。ntp提供的中国地区的时间同步服务器域名列表：

<https://www.pool.ntp.org/zone/cn>

配置 `/etc/sysconfig/ntpd` 文件

```
# Drop root to id 'ntp:ntp' by default.
OPTIONS="-u ntp:ntp -p /var/run/ntpd.pid"
# 设置是否同步到硬件时间
SYNC_HWCLOCK=yes
# Additional options for ntpdate 默认选项为-g
NTPDATE_OPTIONS="-g"
```

### 开启ntpd服务

```bash
systemctl start ntpd.service
```

### 客户端安装使用办法

方法一：启动 `ntp` 服务，修改 server 即上级服务器指定为刚配置好的 ntp 服务器 IP 地址，启动服务即可自动维护时间;

方法二：编写 `ntpdate` 脚本，放入定时任务;

如果客户端没有 `ntpdate` 命令，可以使用 `yum` 安装

```bash
yum install ntpdate
```

补充：可以使用 `yum provides ntpdate`查看 `ntpdate` 命令安装包名称

一个客户端脚本例子：

```bash
#!/bin/bash
#设置上级ntp服务器的IP地址替换ntp_server
ntp_server="ntp.epurs.com"
# 尝试同步ntp_server，并修改硬件时间
/usr/sbin/ntpdate ntp_server && /usr/sbin/hwclock -w ||exit 1
# 添加定时任务，每五分钟同步一次
grep -q hwcklock /var/spool/cron/root &>/null || /bin/echo '*/5 * * * * /usr/sbin/ntpdate ntp_server && /usr/sbin/hwclock -w &>/dev/null' >>/var/spool/cron/root
```

## chrnoy时钟同步服务

安装同上，直接`yum install chrnoy`即可，配置办法很简单，同样最简单的只需要修改 server 即可。需要注意的是，客户端不能使用`ntpdate`进行同步了，需要同样安装`chrnoy`软件，修改配置文件`/etc/chrony.conf`中 server 指定配置的 server ip。可能这也是很多人不能接受新软件的问题，配置使用更麻烦，不便于管理。

`chrnoy`和`ntpd`相比优势在哪里？

- 更快的同步只需要数分钟而非数小时时间，从而最大程度减少了时间和频率误差，这对于并非全天 24 小时运行的台式计算机或系统而言非常有用。
- 能够更好地响应时钟频率的快速变化，这对于具备不稳定时钟的虚拟机或导致时钟频率发生变化的节能技术而言非常有用。
- 在初始同步后，它不会停止时钟，以防对需要系统时间保持单调的应用程序造成影响。
- 在应对临时非对称延迟时（例如，在大规模下载造成链接饱和时）提供了更好的稳定性。
- 无需对服务器进行定期轮询，因此具备间歇性网络连接的系统仍然可以快速同步时钟。

参考文档：[chrony官方手册](https://chrony.tuxfamily.org/manual.html)



## **ntp相关常见命令详解**

**ntp启动参数配置文件**

上面提到了如果想要让 `ntp` 同时同步硬件时间，可以设置 `/etc/sysconfig/ntpd` 文件，对于这个文件还有很多设置可以添加。

有时会在 messages 日志里看到 NTP 进程正在做 DELETING 动作，虽然 NTP 进程并不会真正删除虚拟网络接口，但这个动作会造成网络短暂不通。红帽官方有个对应的解决办法是：就是在这个文件里增加-L参数。

ntpd 服务的方式，又有两种策略，一种是平滑、缓慢的渐进式调整（adjusts the clock in small steps所谓的微调）；一种是步进式调整（跳跃式调整）。两种策略的区别就在于，微调方式在启动NTP服务时加了个 `-x` 的参数，而默认的是不加 `-x`参数。假如使用了 `-x` 选项，那么 ntpd 只做微调，不跳跃调整时间，但是要注意，-x参数的负作用：当时钟差大的时候，同步时间将花费很长的时间。`-x` 也有一个阈值，就是 600s，当系统时钟与标准时间差距大于 600s 时，ntpd 会使用较大“步进值”的方式来调整时间，将时钟“步进”调整到正确时间。假如不使用 `-x` 选项，那么ntpd 在时钟差距小于128ms时，使用微调方式调整时间，当时差大于128ms时，使用“跳跃”式调整。

上述这两种方式在本地时钟与远端的 NTP 服务器时钟相差大于 1000s 时，ntpd 会停止工作。在启动 NTP 时加了参数 `-g` 就可以忽略 1000s 的问题，新版 ntpd 服务默认文件是有 `-g` 参数的。



##### ntpstat 可以查看ntp服务运行状况

```bash
# ntpstat
unsynchronised
  time server re-starting
   polling server every 8 s

```

这种情况可能是 server 配置的服务器问题，不过大多数情况下都需要一段联机时间，鸟哥书中提到大概需要15min，我测试的实际情况大概在5min。这同时也提醒我们，如果配置 ntp 服务器，要提前使用 `ntpdate` 进行同步时间，再开启 ntpd 服务。

```bash
# ntpstat
synchronised to NTP server (182.92.12.11) at stratum 3
   time correct to within 1209 ms
   polling server every 64 s

```
这里是配置成功的情况

##### ntpdate 手动同步时间

`ntpdate -d` 参数可以查看详细的同步信息，找出同步失败的问题。

`ntpq -p` 可以查看当前服务器上级 ntp 服务器连接情况

```bash
# ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 time6.aliyun.co 10.137.38.86     2 u   18   64    7  597.977  289.135 128.856
 time4.aliyun.co 10.137.38.86     2 u   18   64    7  281.075  127.728 167.399
 time5.aliyun.co 10.137.38.86     2 u   20   64    7  584.615  273.673 126.084

```

上面部分列的解释:

`remote` – 本机和上层ntp的ip或主机名，“+”表示优先，“*”表示次优先

`refid` – 上一层ntp主机地址

`st – stratum` 阶层。remote远程服务器的级别.由于NTP是层型结构,有顶端的服务器,多层的Relay Server再到客户端.所以服务器从高到低级别可以设定为1-16.为了减缓负荷和网络堵塞,原则上应该避免直接连接到级别为1的服务器的.

`when` – 多少秒前曾经成功同步过时间，单位 s 。

`poll` – 下次更新在多少秒后。本地机和远程服务器多少时间进行一次同步(单位为秒).在一开始运行NTP的时候这个poll值会比较小,那样和服务器同步的频率也就增加了,可以尽快调整到正确的时间范围，之后poll值会逐渐增大,同步的频率也就会相应减小。

`reach` – 已经向上层 ntp 服务器要求更新的次数。这是一个八进制值,用来测试能否和服务器连接.每成功连接一次它的值就会增加。

`delay` – 网络延迟。从本地机发送同步要求到 ntp 服务器的 round trip time 。

`offset` – 时间补偿。主机通过 NTP 时钟同步与所同步时间源的时间偏移量，单位为毫秒（ms）。offset越接近于0,主机和ntp服务器的时间越接近。

`jitter` – 系统时间与 bios 时间差。它统计了在特定个连续的连接数里offset的分布情况.简单地说这个数值的绝对值越小，主机的时间就越精确。

`ntptrace` 可以查看当前服务器同上层服务器间的联系

```bash
ntptrace -n 127.0.0.1
```

## 安全性设定

iptables配置部分：

```bash
iptables -A INPUT -p udp -s 192.168.1.0/24 --dport 123 -j ACCEPT
```
