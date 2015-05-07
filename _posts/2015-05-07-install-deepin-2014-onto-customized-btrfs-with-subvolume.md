---
title: 在自定义的 btrfs+subvolume 上安装 Deepin 2014
author: MattD
layout: post
permalink: /2015/05/07/install-deepin-2014-onto-customized-btrfs-with-subvolume.html
categories: Linux
tags: [btrfs, Deepin, subvolume]
---
本文参考自 woodelf 在 Deepin 论坛发的[帖子](http://bbs.deepin.org/forum.php?mod=viewthread&tid=16324)。由于 Deepin 2014 版的安装程序脚本有了较大变动，因此跟原帖相比，还需要进行许多额外操作。

另外，Deepin 2014.3 的官方 ISO 不再提供 LiveCD 功能，因此需要借助其他发行版的 Live 介质。Ubuntu 系的 LiveCD 是个不错的选择。

<!-- more -->

进入 Live 环境后，首先需要安装必要的工具：

{% highlight bash %}
# apt-get install btrfs-tools squashfs-tools
{% endhighlight %}

根据你自己的需要随意划分分区，建议将 `/boot` 单独分区。由于我们要采用 subvolume，因此只需要一个 btrfs 分区即可。另外，由于 btrfs 尚未支持 swap file，因此如果你的内存比较小，或者需要使用休眠功能，则必须再单独划分一个 swap 分区。

我自己的分区情况是：第一分区为一个 1MiB 大小的 BIOS 启动分区（因为我用的是 GPT，但是我的主板固件不支持 UEFI），第二分区为 512MiB 的 `/boot` 分区，第三分区为 swap 分区，第四分区为 btrfs 分区。

把 btrfs 分区挂载到 `/mnt`，并指定挂载参数（Ubuntu 的安装程序习惯把目标分区挂载到 `/target`）：

{% highlight bash %}
# mount -o defaults,compress=lzo,space_cache,autodefrag /dev/sda4 /mnt
{% endhighlight %}

接着，先创建一个 subvolume，作为以后 Deepin 的 /，名字可以随意命名：

{% highlight bash %}
# cd /mnt
# btrfs subvolume create deepin
{% endhighlight %}

然后按照喜好，任意创建其他 subvlume，作为以后 / 下的其他挂载点。这里，我们有两种 subvolume 方案，一种是跟传统目录树一样的树形结构，另一种是创建多个平级的 subvolume。这两种方案各有利弊，我先分别举例说明一下。

树形结构：

{% highlight bash %}
cd /mnt
btrfs subvolume create deepin/home
btrfs subvolume create deepin/opt
btrfs subvolume create deepin/srv
btrfs subvolume create deepin/var
{% endhighlight %}