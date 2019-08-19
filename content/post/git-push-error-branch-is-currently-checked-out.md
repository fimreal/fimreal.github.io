---
title: "记一次 git push 失败解决办法"
date: 2019-04-16T08:36:34Z
lastmod: 2019-04-16T11:50:57Z
draft: false
keywords: ["Git"]   
description: "记一次 git push 失败解决办法"
tags: ["Git"]
categories: ["Git"]
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
`git push` 总是失败，加了`--force`参数也不成功，命令行执行时会提示：`branch is currently checked out`。
原因是什么？如何解决呢？

<!--more-->

`git push` 出现报错，即时使用 `--force` 参数也不成功，会出现类似下面的报错：
```
[remote rejected] master -> master (branch is currently checked out)
remote: error: refusing to update checked out branch: refs/heads/master
remote: error: By default, updating the current branch in a non-bare repository
remote: error: is denied, because it will make the index and work tree inconsistent
remote: error: with what you pushed, and will require 'git reset --hard' to match
remote: error: the work tree to HEAD.
remote: error:
remote: error: You can set 'receive.denyCurrentBranch' configuration variable to
remote: error: 'ignore' or 'warn' in the remote repository to allow pushing into
remote: error: its current branch; however, this is not recommended unless you
remote: error: arranged to update its work tree to match what you pushed in some
remote: error: other way.
remote: error:
remote: error: To squelch this message and still keep the default behaviour, set
remote: error: 'receive.denyCurrentBranch' configuration variable to 'refuse'.
To ***
 ! [remote rejected] master -> master (branch is currently checked out)
error: failed to push some refs to '***'
```
原因：
其实是因为远端仓库当前所在分支也是 `master` 导致 `push` 时会出现错误提示。

**解决办法**：
1. 登陆代码仓库，修改配置文件 `.git/config`，添加下面内容：
```
[receive]
denyCurrentBranch = ignore
```
2. 执行 `git push`
3. 成功后，在 repo 端执行：
```bash
git reset --hard
```
即可完成更新步骤。
不执行这步的话，在仓库端使用 `git status` 会看到新添加文件在当前工作区处于 deleted 状态，所以实际不影响文件 push。
**参考链接**
https://www.cnblogs.com/abeen/archive/2010/06/17/1759496.html
