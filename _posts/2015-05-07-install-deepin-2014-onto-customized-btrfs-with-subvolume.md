---
title: Install Deepin 2014 onto Customized btrfs with Subvolume
author: MattD
layout: post
permalink: /2015/05/07/install-deepin-2014-onto-customized-btrfs-with-subvolume.html
categories: Linux
tags: [btrfs, Deepin, subvolume]
---
本文参考自 woodelf 在 Deepin 论坛发的[帖子](http://bbs.deepin.org/forum.php?mod=viewthread&tid=16324)。由于 Deepin 2014 版的安装程序脚本有了较大变动，因此跟原帖相比，还需要进行许多额外操作。

另外，Deepin 2014.3 的官方 ISO 不再提供 LiveCD 功能，因此需要借助其他发行版的 Live 介质。Ubuntu 系的 LiveCD 是个不错的选择。

<!-- more -->

进入 Live 环境后，首先需要安装必要的工具

{% highlight bash %}
# apt-get install btrfs-tools squashfs-tools
{% endhighlight %}