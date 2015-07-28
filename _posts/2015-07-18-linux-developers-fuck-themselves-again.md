---
title: Linux 开发人员再次扯到了自己蛋
author: MattD
layout: post
permalink: /2015/07/18/linux-developers-fuck-themselves-again.html
categories: Linux
tags: [kernel, Linux]
---
还记得五月份的那篇文章么？嘿嘿，Linux 开发人员果然不负众望，再次在 4.2 内核的 rc 版本中扯到自己蛋了。

这次的问题和之前一模一样，只不过这次出问题的是 flush\_workqueue 这个符号。因此，我已经无力吐槽，于是干脆就自己写了个 ebuild，直接自动打好内核补丁，来安装最新版 rc 内核。

稍后会把新的 ebuild 提交到我的 github，敬请拭目以待。

<!-- more -->

另外，我也会把新写的几个 ebuild 提交上去，只不过，安装失败的后果自负，我只保证 rc 版内核没问题。

Nvidia 用户万岁！每次都能用黑科技来解决内核模块编译失败问题，这感觉真是太爽了。
