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

进入 Live 环境后，首先需要安装必要的工具：

{% highlight bash %}
# apt-get install btrfs-tools squashfs-tools
{% endhighlight %}

根据你自己的需要随意划分分区，建议将 `/boot` 单独分区。由于我们要采用 subvolume，因此只需要一个 btrfs 分区即可。另外，由于 btrfs 尚未支持 swap file，因此如果你的内存比较小，或者需要使用休眠功能，则必须再单独划分一个 swap 分区。

<!-- more -->

我自己的分区情况是：第一分区为一个 1MiB 大小的 BIOS 启动分区（因为我用的是 GPT，但是我的主板固件不支持 UEFI），第二分区为 512MiB 的 `/boot` 分区（ext2），第三分区为 swap 分区，第四分区为 btrfs 分区。

把 btrfs 分区挂载到 `/mnt`，并指定挂载参数（我这里指定了 lzo 压缩、自动后台整理碎片以及空闲空间缓存加速。Ubuntu 的安装程序习惯把目标分区挂载到 `/target`）：

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
# cd /mnt
# btrfs subvolume create deepin/home
# btrfs subvolume create deepin/opt
# btrfs subvolume create deepin/srv
# btrfs subvolume create deepin/var
{% endhighlight %}

平级结构：

{% highlight bash %}
# cd /mnt
# btrfs subvolume create home
# btrfs subvolume create opt
# btrfs subvolume create srv
# btrfs subvolume create var
{% endhighlight %}

这两种方案各有利弊，具体来说：

第一种方案挂载比较方便，只需要挂载最上面一级的 subvolume，位于它下方的二级 subvolume 就能自动挂载了，不需要再手动挂载每个 subvolume。在生成快照的时候，也可以递归进行 snapshot 操作。但是，这种方案无法为每个二级 subvolume 单独设定挂载选项，设定了也没用，因为二级 subvolume 的挂载选项只能从上级 subvolume 继承而来。

第二种方案就要灵活得多，每个 subvolume 可以单独设置挂载选项，但是如果要维护的时候挂载过程比较繁琐，尤其是分了较多的 subvolume 的话。另外，这种方案无法实现递归快照，即使你把每个 subvolume 按照属性结构相应挂载了。

将创建好的 subvolume 重新挂载到对应的挂载点：

如果你采用的是树形结构，那么就可以跳过这一步。如果采用平级结构，那么就请按照以下步骤重新挂载 subvolume。

{% highlight bash %}
# mount -o remount,subvol=home,defaults,compress=lzo,space_cache,autodefrag /dev/sda4 /mnt/deepin/home
# mount -o remount,subvol=opt,defaults,compress=lzo,space_cache,autodefrag /dev/sda4 /mnt/deepin/opt
# mount -o remount,subvol=srv,defaults,compress=lzo,space_cache,autodefrag /dev/sda4 /mnt/deepin/srv
# mount -o remount,subvol=var,defaults,compress=lzo,space_cache,autodefrag /dev/sda4 /mnt/deepin/var
{% endhighlight %}

挂载 `/boot`：

{% highlight bash %}
# mkdir /mnt/deepin/boot
# mount /dev/sda2 /mnt/deepin/boot
{% endhighlight %}

把 Deepin 2014 的安装介质挂载到 `/mnt2`（这里我以 ISO 文件为例）：

{% highlight bash %}
# mkdir /mnt2
# mount /path/to/deepin.iso /mnt2
{% endhighlight %}

正式安装 Deepin 2014（其实就是解压 squashfs，安装程序的本质也是如此）。注意，由于 Deepin 2014.3 的 ISO 集成了多语言包和两套可选的 Office 套件（WPS 和 LibreOffice），因此需要单独把语言包和你喜欢的 Office 套件包也解压到目标分区。这里我以简体中文语言包和 LibreOffice 为例：

{% highlight bash %}
# unsquashfs -f -d /mnt/deepin /mnt2/casper/filesystem.squashfs
# unsquashfs -f -d /mnt/deepin /mnt2/casper/overlay-deepin-language-pack-zhcn.squashfs
# unsquashfs -f -d /mnt/deepin /mnt2/casper/overlay-deepin-office-libreoffice.squashfs
{% endhighlight %}

等待解压过程完成，系统就算是安装好了。接着，我们需要 chroot 到安装好的系统里面去，进行一些必要的配置：

{% highlight bash %}
# cd /mnt/deepin
# mount --bind /dev dev
# mount --bind /dev/pts dev/pts
# mount --bind /proc proc
# mount --bind /sys sys
# chroot /mnt/deepin /bin/bash
{% endhighlight %}

首先来配置 `/etc/fstab`，这个文件事关重大。

先用 btrfs 命令查看下各个 subvolume 对应的 id，稍后会有用：

{% highlight bash %}
# btrfs subvolume list /mnt
{% endhighlight %}

接着再用 blkid 命令查看并记下各个分区对应的 UUID。注意，我们需要的是 UUID，而不是 UUID_SUB。然后，按照以下格式编写 `/etc/fstab`：

{% highlight bash %}
# UNCONFIGURED FSTAB FOR BASE SYSTEM
UUID=ffd7019b-d772-4069-821e-4fc74c8b22a9  /boot  ext2  defaults,noatime  1 2
UUID=01dc1816-8aa5-4490-b052-68c22f69a11c  none   swap  sw                0 0
UUID=5713767f-e3f3-4871-8db9-9f5a2c3c37df  /      btrfs defaults,subvolid=256,compress=lzo,space_cache,autodefrag  0 0
UUID=5713767f-e3f3-4871-8db9-9f5a2c3c37df  /home  btrfs defaults,subvolid=258,compress=lzo,space_cache,autodefrag  0 0
UUID=5713767f-e3f3-4871-8db9-9f5a2c3c37df  /opt   btrfs defaults,subvolid=259,compress=lzo,space_cache,autodefrag  0 0
UUID=5713767f-e3f3-4871-8db9-9f5a2c3c37df  /srv   btrfs defaults,subvolid=260,compress=lzo,space_cache,autodefrag  0 0
UUID=5713767f-e3f3-4871-8db9-9f5a2c3c37df  /var   btrfs defaults,subvolid=262,compress=lzo,space_cache,autodefrag  0 0
{% endhighlight %}

里面的 subvolid 就是之前查看到的每个 subvolume 的 id。如果你的分区经常变动，建议把 UUID 换成分区 label，尽量不要用 `/dev/sdXY` 这样的设备名，这是最不可靠的。

然后修改 `/etc/hostname`，如果不存在这个文件就新建一个，内容只需要写上你喜欢的主机名就行了。

接着修改默认 locale，如果不设置，系统就会把 locale 自动 fallback 为 POSIX。创建 `/etc/default/locale` 文件，写入以下内容：

{% highlight bash %}
LANG="zh\_CN.UTF-8"
LANGUAGE="zh\_CN.UTF-8"
{% endhighlight %}

还要修改 `/etc/default/useradd`，以指定新用户的默认 shell，把 `SHELL=/bin/sh` 这一行改为 `SHELL=/bin/zsh`。

添加新用户，我这里已经指定了所有必要的组：

{% highlight bash %}
# useradd -s /bin/zsh -m -G users,cdrom,floppy,audio,video,plugdev,sambashare,adm,wheel,lp,netdev,scanner,lpadmin,sudo,storage,network USERNAME
{% endhighlight %}

设置 root 和用户密码：

{% highlight bash %}
# passwd root
# passwd USERNAME
{% endhighlight %}

通过 `visudo` 命令，把你的用户加入 sudoer 列表。在 `root ALL=(ALL:ALL) ALL` 下面添加一行：

{% highlight bash %}
USERNAME ALL=(ALL:ALL) ALL
{% endhighlight %}

把 grub 安装到 `/dev/sda`，并更新 initramfs，忽略过程中的错误提示。这里我们还不能更新 grub.cfg，因为 grub 识别不了根分区（这里仅以 BIOS 版 grub 为例，UEFI 版类似）：

{% highlight bash %}
# grub-install /dev/sda
# update-initramfs -u -k all
{% endhighlight %}

编写 `/etc/hosts`，内容如下，否则系统会出现各种奇怪的问题。里面的 Deepin 是我设置的主机名：

{% highlight bash %}
127.0.0.1	localhost
127.0.1.1   Deepin
# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
{% endhighlight %}

将 `/etc/lightdm/lightdm.conf` 修改为如下内容，否则系统启动后会出现空白的语言选择界面，无法登录：

{% highlight bash %}
[SeatDefaults]
greeter-session=lightdm-deepin-greeter
user-session=deepin
{% endhighlight %}

然后卸载 `/mnt` 下挂载的所有文件系统，并且从 `/dev/sda` 引导。由于之前没有更新 grub.cfg，因此此处会进入 grub rescue。不着急，依次输入以下命令即可正常引导：

{% highlight bash %}
grub> insmod part_gpt
grub> insmod ext2
grub> set root=(hd0,gpt2)
grub> linux /vmlinuz-3.13.0-51-generic root=/dev/sda4 quiet rootflags=subvol=deepin nosplash
grub> initrd /initrd-3.13.0-51-generic
grub> boot
{% endhighlight %}

注意，set root 那一行指的是 grub 的 root，即 `/boot` 或 `/boot/EFI` 所在的分区，linux 那一行的 root 才是指的根分区。另外，rootflags=subvol=deepin 内核参数中，后面的 deepin 需要改为你先前创建的根 subvolume 名称。如果你是 MBR 分区表，那么分别把第一行和第三行中的 gpt 改为 msdos 即可。

不出意外，应该可以顺利登录系统了，效果和安装程序安装的系统一样。记得生成一份 grub.cfg 配置文件：

{% highlight bash %}
# update-grub
{% endhighlight %}

到此就全部结束了，这种安装方法只是一个示例，通过这种方法可以把 Deepin 灵活安装在很多复杂的分区方案上，如 LVM+LUKS、dmraid+LUKS 等。
