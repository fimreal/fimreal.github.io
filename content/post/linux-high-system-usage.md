---
title: "记一次 system usage 占用高的解决过程"
date: 2022-12-26T11:50:09+08:00
lastmod: 2022-12-26T11:50:09+08:00
draft: false
keywords: ["perf","top"]
description: "这些天遇到一个奇怪问题，Linux 服务器负载持续在6以上，system usage 占用超过50%，但是通过 找不到高 cpu 占用的进程"
tags: ["linux"]
categories: ["linux"]
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
最近有台 2c 服务器负载经常维持在6-9，每天跑定时任务时间还会因为 cpu 占用过高报警，登上去 `ps aux | sort -nk3`查看进程 cpu 占用都很低，top 命令可以直观看到进程 cpu 占用也只有不到 30%，但是 sy 占用高达60%，显示系统负载非常高。

<!--more-->

## 异常服务器状态

top 命令部分结果如下，可以看到 us 占用并不高，问题应该是出在 sy 占用

```bash
top - 08:38:18 up 374 days, 16:40,  1 user,  load average: 7.19, 7.18, 7.31
Tasks: 173 total,   5 running, 167 sleeping,   0 stopped,   1 zombie
%Cpu(s): 26.3 us, 57.4 sy,  0.0 ni, 16.1 id,  0.0 wa,  0.0 hi,  0.2 si,  0.0 st
KiB Mem :  8010460 total,   859648 free,  4964672 used,  2186140 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  2650992 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
11325 root      10 -10  189.2m  68.2m   6.1m S  23.2  0.9   7163:02 /usr/local/aegis/aegis_client/aegis_11_41/AliYunDunMonitor
11461 root      20   0 1225.2m 396.4m   3.5m S  12.9  5.1  30737:18 sdsvrd -d
 4150 root      10 -10 1028.6m  37.3m   1.2m S   1.3  0.5   1771:11 /usr/local/aegis/alihips/AliHips
```

查看系统日志，只有潦潦一些 sshd 登录记录。

内存很空闲，磁盘也完全不紧张，没有读写压力。

## 排查解决过程

通过 google 确实找到了不错的思路，参考下文一个 php 博主的压力测试帖子，提到了使用 perf 工具排查系统调用损耗。

https://learnku.com/articles/39516

安装 perf 工具，参考文中办法生成报告。

```bash
# 切换到 root
# sudo su -
$ cd /tmp
# 等待15s 左右，ctrl+c 打断
$ perf record -g
^C[ perf record: Woken up 77 times to write data ]
[ perf record: Captured and wrote 19.891 MB perf.data (102521 samples) ]
# 读取记录
$ perf report
```

记录部分如下，在终端中按 e 可以展开，按 d 合上：

```bash
Samples: 102K of event 'cpu-clock', Event count (approx.): 25630250000
  Children      Self  Command          Shared Object                      Symbol
+   19.77%     0.01%  ps               [kernel.kallsyms]                  [k] system_call_fastpath
+   15.91%     0.02%  swapper          [kernel.kallsyms]                  [k] cpu_startup_entry
+   15.56%     0.02%  swapper          [kernel.kallsyms]                  [k] arch_cpu_idle
+   15.54%     0.04%  swapper          [kernel.kallsyms]                  [k] default_idle
+   15.49%    15.49%  swapper          [kernel.kallsyms]                  [k] native_safe_halt
+   11.01%     0.00%  ps               [unknown]                          [k] 0x0000000000000800
+    8.70%     0.04%  ps               libc-2.17.so                       [.] __GI___libc_read
+    8.62%     0.02%  ps               [kernel.kallsyms]                  [k] sys_read
+    8.57%     0.03%  ps               [kernel.kallsyms]                  [k] vfs_read
+    8.54%     0.00%  swapper          [kernel.kallsyms]                  [k] start_secondary
+    7.82%     0.92%  killmft.sh       [kernel.kallsyms]                  [k] ReleaseProcessNotify
+    7.50%     0.04%  ps               [kernel.kallsyms]                  [k] seq_read
+    7.40%     0.00%  swapper          [kernel.kallsyms]                  [k] x86_64_start_kernel
+    7.40%     0.00%  swapper          [kernel.kallsyms]                  [k] x86_64_start_reservations
+    7.40%     0.00%  swapper          [kernel.kallsyms]                  [k] start_kernel
+    7.40%     0.00%  swapper          [kernel.kallsyms]                  [k] rest_init
+    7.25%     0.00%  killmft.sh       libc-2.17.so                       [.] __execve
+    7.17%     0.00%  killmft.sh       [kernel.kallsyms]                  [k] stub_execve
+    7.17%     0.00%  killmft.sh       [kernel.kallsyms]                  [k] sys_execve
+    7.16%     0.00%  killmft.sh       [kernel.kallsyms]                  [k] do_execve_common.isra.25
+    6.88%     0.00%  killmft.sh       [kernel.kallsyms]                  [k] search_binary_handler
+    6.75%     0.02%  ps               [kernel.kallsyms]                  [k] proc_single_show
+    6.26%     0.00%  killmft.sh       [kernel.kallsyms]                  [k] LoadBinaryNotify
+    6.25%     0.01%  killmft.sh       [kernel.kallsyms]                  [k] ReleaseDebugPerf
+    6.24%     3.57%  killmft.sh       [kernel.kallsyms]                  [k] FnMatch
+    5.91%     0.03%  ps               libc-2.17.so                       [.] __GI___libc_open
+    5.88%     0.16%  ps               [kernel.kallsyms]                  [k] InitFtraceHook
+    5.79%     0.00%  killmft.sh       [unknown]                          [k] 0x00000000022aedb0
+    5.67%     0.00%  killmft.sh       [unknown]                          [k] 0x00000000022aed30
+    5.20%     0.11%  killmft.sh       [kernel.kallsyms]                  [k] BuildProcOwnerChain
+    5.07%     0.16%  ps               [kernel.kallsyms]                  [k] proc_pid_status
+    4.10%     0.00%  AliYunDunMonito  libaqsUtil.so.1                    [.] aqs::CThreadUtil::ThreadFunc
+    4.06%     0.01%  ps               [kernel.kallsyms]                  [k] sys_open
+    3.82%     0.04%  ps               [kernel.kallsyms]                  [k] do_sys_open
+    3.67%     0.00%  ps               [unknown]                          [.] 0000000000000000
+    3.65%     0.20%  killmft.sh       [kernel.kallsyms]                  [k] MatchOpe
+    3.60%     1.18%  ps               [kernel.kallsyms]                  [k] vsnprintf
```

可以明显发现 ps 命令占用特别高，而且重复出现了一个脚本名字 `killmft.sh` 。`swapper` 是内核空闲时间跑的，可以忽略。

现在目标明确了，大概猜测一下有某个进程高频率调用`killmft.sh`这个脚本，而脚本里用到了 ps 命令，造成内核不停在查询进程信息，表现就是看不到高 cpu 进程存在，但是内核又占用了大量 cpu 时间。

查一下`killmft.sh`相关进程：

```bash
$ ps aux | grep killmft
xxx  12635  0.0  0.0 113412   744 ?        Ss   3月07  35:53 /bin/bash /home/xxx/killmft.sh
xxx  12829  0.0  0.0 112828   980 pts/0    S+   09:07   0:00 grep --color=auto killmft
xxx  12830  0.0  0.0 113284   556 ?        S    09:07   0:00 /bin/bash /home/xxx/killmft.sh
xxx  22615  0.0  0.0 113412   760 ?        S    3月07  38:47 sh killmft.sh
xxx  22942  0.0  0.0 113412   744 ?        Ss   3月07  35:54 /bin/bash /home/xxx/killmft.sh
xxx  24778  0.6  0.0 113284   616 ?        Ss   3月07 2770:30 /bin/bash /home/xxx/killmft.sh
```

竟然发现多条进程都在跑，脚本内容：

```bash
$ cat /proc/22615/fd/255
#!/bin/bash
while true
do
  PIDS=`ps uax|grep sclient.jar|egrep 'bak|0131000001'|grep -v grep |awk '{split($10,a,":");if (a[1] >= 1 ) print $2}'`
  for pid in ${PIDS[@]}
  do
    kill $pid
  done
  sleep 10
done
```

这里有个小技巧，fd/255 通常对应的是执行脚本。

脚本内容很简单，检查 mft 进程，执行超过 1分钟就杀掉，用 while true 保证 10s 执行一次。

脚本本身10s 一次频率还能接受，但是上面进程里同时发现了 5 条执行记录，平均 2s 就要执行一次 ps 命令，或许一台轻量的 2c 服务器就是这样被拖垮的。

于是我主动 kill 掉其中4个进程，发现负载快速降低到了 0.x，那么就可以肯定罪魁祸首就是它。

## 一些想法

最近也是遇到光大 mft 客户端升级，免不了要更新原来的一些 bash 脚本。银行的客户端怎么说呢，更新次数不多，但问题不少。我不好说人银行技术怎么样，毕竟下决心更新一个近20年的服务已经很让人感动了，战战兢兢改脚本只能说是我的问题。

bash 这种脚本化语言，很容易出现不按预期执行的情况，比如 bash 搞成 sh，test 判断出错，甚至是保留字符被认为是命令等等，问题千奇百怪。bash 作为运维处理简单的项目时方便好用，但是写的越长，实现功能越复杂，hack 用法越多，越容易出现问题。所以说写 bash 脚本时，不能贪图简单像命令行一行就想解决问题。写的代码多，代码冗余并不代表质量不好，做好异常判断很重要。

脚本嘛，修修补补是不可避免的。文中出现的问题其实很好解决，在代码里生成一个 pid 文件避免重复执行即可。

作运维总结经验很重要，这还是第一次成功排查到 sy 占用高问题的原因并且成功解决了。自己总结的笔记今年又厚了好多，软件也变得越来越卡，要找个机会把东西都归档一下，或者整理下放在github/博客里记录了。
