---
title: 从当前 Linux 宿主安装 Arch Linux
author: MattD
layout: post
permalink: /2015/09/18/install-arch-linux-from-a-running-linux-host.html
categories: Linux
tags: [Arch, Linux]
---
前两天看到了竹子君在深度论坛发的 VM 安装 Arch Linux 教程，刚好我也顺便整理下之前自己实践过的一些经验吧。

首先要说明的是，只要有一个运行中的 Linux 宿主，无论是物理安装好的 Linux 还是 LiveCD 环境，理论上都可以通过目标发行版的包管理器来完整安装出一个全新的目标发行版。

当然，前提是你能在宿主上想办法安装目标发行版的包管理，且目标发行版的包管理支持指定安装目录。

其实之前我也尝试过以运行中的 Gentoo 为宿主，通过 zypper 从头安装了一个完整的 openSUSE，自然成功了。原理一样，这里就以安装 Arch 为例说明，其他发行版在此基础上变通即可。

<!-- more -->

第一步，自然是先规划好目标分区，然后挂载到 `/mnt`，这里就不展开了，没什么好说的。

别忘了把 `/dev`、`/dev/pts`、`/proc`、`/sys` 等必要的挂载点也以 --bind 模式挂载过去。`/etc/resolv.conf` 也要复制过去，否则后面 pacman 无法解析域名。

第二步，在当前宿主中安装 pacman，并配置好 mirror 和 repo 等参数。注意，SigLevel 必须设置为 Never，宿主不支持 pacman 的 GPG Key 机制，会导致 pacman 报错。

第三步，创建 pacman 正常运行所必要的目录，这步很关键，否则 pacman 也会报错。

{% highlight bash %}
# mkdir -p /var/lib/pacman
# mkdir -p /mnt/var/lib/pacman
{% endhighlight %}

第四步，同步软件源数据，指定 `/mnt` 作为 rootfs，注意其中的 -r 参数。如果这一步没问题，可以正常同步完，那就可以继续下一步了。

{% highlight bash %}
# pacman -Syy -r /mnt
{% endhighlight %}

第五步，正式把 base 和 base-devel 安装到 `/mnt`。

{% highlight bash %}
# pacman -S base base-devel -r /mnt
{% endhighlight %}

如果第五步顺利完成，那么主要步骤其实就结束了，后面的 chroot 部分完全可以参照 Arch Wiki。但是需要注意一点，由于宿主没有 AIS，因此目标发行版的 `/etc/fstab` 是需要手写的。还有就是，务必注意 mkinitcpio 的配置，这会影响到安装好的 Arch Linux 能否正常启动。

通过这方法，理论上可以毫无压力随心所欲以任意运行中的发行版或 LiveCD 来从头安装其他发行版。当时我用 zypper 从头安装 openSUSE 也是按照这套路走下来的，虽然花了点时间去研究 openSUSE 的 pattern 打包机制。

另外，像 Debian 和 Fedora 提供了 debootstrap 和 febootstrap 这样的脚本，可以更自动地实现这种安装方式，有机会也可以尝试下。

PS：本文其实没什么技术含量，了解了原理就知道这个过程非常简单。就纯当这是一篇凑数的文章吧。
