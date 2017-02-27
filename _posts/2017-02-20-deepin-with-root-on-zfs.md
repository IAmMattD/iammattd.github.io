---
title: 将 Deepin 安装到 Root On ZFS
author: MattD
layout: post
permalink: /2017/02/20/deepin-with-root-on-zfs.html
categories: Linux
tags: [ZFS, Deepin]
---
之前曾经在 Deepin 2014 尝试过 Root On ZFS，可惜失败了。最近心血来潮，打算把 Deepin 15 安装到 ZFS Root，在解决了几个小问题以后，成功。

现把流程记录如下，以作备忘。

首先需要进入 Live 环境（很简单，在 ISO 引导界面编辑一下菜单项，把 `livecd-installer` 内核参数改为 `livecd` 就行了）。

<!-- more -->

在安装 ZFS 模块之前，首先需要进行 dkms 配置，否则 ZFS 模块因为缺少必要的头文件，会安装失败。需要将以下内容写入 `/etc/dkms/spl.conf`。

{% highlight bash %}
POST_INSTALL="scripts/dkms.postbuild -a ${arch} -k ${kernelver} -v ${PACKAGE_VERSION} -n ${PACKAGE_NAME} -t ${dkms_tree}"
{% endhighlight %}

安装一些必要的工具：

{% highlight bash %}
# apt update
# apt install zfs-dkms
{% endhighlight %}

等待安装完成，然后启用一下刚安装的 ZFS 模块，否则后续步骤无法进行。

{% highlight bash %}
# modprobe zfs
{% endhighlight %}

然后开始按照需要创建分区和 zpool。为了以防万一，我将 `/boot` 单独分区，剩下的空间创建一个分区，并全部留给 zpool。另外，建议使用磁盘 ID 而不是块设备名来创建 zpool。具体的参数作用可以自己查找手册了解。

{% highlight bash %}
# zpool create -f -o ashift=12 -o cachefile= -O normalization=formD -O compression=lz4 -m none -R /mnt rpool /dev/disk/by-id/scsi-SATA_disk1
{% endhighlight %}

此时，我创建的名为 `rpool` 的 zpool 已经挂载到了 `/mnt`。然后，根据需要创建其它 ZFS dataset。

{% highlight bash %}
# zfs create -o mountpoint=none rpool/ROOT
# zfs create -o mountpoint=/ rpool/ROOT/deepin
# zfs create -o mountpoint=/home rpool/HOME
# zfs create -o mountpoint=none rpool/DEEPIN
# zfs create -o mountpoint=/usr/src rpool/DEEPIN/src
# zfs create -o mountpoint=/srv rpool/DEEPIN/srv
{% endhighlight %}

然后，设置下 ZFS 属性。

{% highlight bash %}
# zpool set bootfs=rpool/ROOT/deepin rpool
{% endhighlight %}

此外，我把 swap 放在 zvol 里面，就省得单独分区了。swap 大小根据需要设定。

{% highlight bash %}
# zfs create -o sync=always -o primarycache=metadata -o secondarycache=none -V 12G rpool/swap
# mkswap -f /dev/zvol/rpool/swap
{% endhighlight %}

挂载 `/boot`。

{% highlight bash %}
# mkdir /mnt/boot
# mount /dev/sda2 /mnt/boot
{% endhighlight %}

准备工作完成了，接下来开始正式安装 Deepin 15（其实就是解压 squashfs，安装程序的本质也是如此）。注意，由于语言包等附加组件是以 overlay 形式打包的，因此需要根据实际情况取舍。

{% highlight bash %}
# unsquashfs -f -d /mnt /lib/live/mount/medium/live/filesystem.squashfs
# unsquashfs -f -d /mnt /lib/live/mount/medium/overlay/overlay-language-pack-zh-hans.squashfs
{% endhighlight %}

等待解压过程完成，系统就算是安装好了。然后把 `zpool.cache` 复制到目标系统中，ZFS 的属性和挂载信息会记录在这里。

{% highlight bash %}
# mkdir -p /mnt/etc/zfs
# cp /etc/zfs/zpool.cache /mnt/etc/zfs/
{% endhighlight %}

接着，我们需要 chroot 到安装好的系统里面去，进行一些必要的配置。

{% highlight bash %}
# cd /mnt
# mount --bind /dev dev
# mount --bind /dev/pts dev/pts
# mount --bind /proc proc
# mount --bind /sys sys
# cp /etc/resolv.conf /mnt/etc/
# chroot /mnt /bin/bash
{% endhighlight %}

首先来配置 `/etc/fstab`，因为 ZFS 的挂载信息全部记录于 `zpool.cache` 中，因此这里我们只需要配置 `/boot` 和 swap 的挂载信息就行了。

{% highlight bash %}
# UNCONFIGURED FSTAB FOR BASE SYSTEM
UUID=ffd7019b-d772-4069-821e-4fc74c8b22a9   /boot   ext4  defaults,noatime  1 2
/dev/zvol/rpool/swap                        none    swap  sw                0 0
{% endhighlight %}

然后修改 `/etc/hostname`，如果不存在这个文件就新建一个，内容只需要写上你喜欢的主机名就行了。

然后添加新用户，按照需求修改所属组。

{% highlight bash %}
# useradd -s /bin/bash -m -G users,cdrom,disk,audio,video,games,plugdev,bluetooth,netdev USERNAME
{% endhighlight %}

设置 root 和用户密码。

{% highlight bash %}
# passwd root
# passwd USERNAME
{% endhighlight %}

通过 `visudo` 命令，把你的用户加入 sudoer 列表。在 `root ALL=(ALL:ALL) ALL` 下面添加一行：

{% highlight bash %}
USERNAME ALL=(ALL:ALL) ALL
{% endhighlight %}

和一开始一样，需要配置下 dkms 配置文件，将以下内容写入 `/etc/dkms/spl.conf`。

{% highlight bash %}
POST_INSTALL="scripts/dkms.postbuild -a ${arch} -k ${kernelver} -v ${PACKAGE_VERSION} -n ${PACKAGE_NAME} -t ${dkms_tree}"
{% endhighlight %}

除此之外，还要把以下内容写入 `/usr/share/initramfs-tools/conf.d/zfs`。

{% highlight bash %}
for x in $(cat /proc/cmdline)
do
    case $x in
        root=ZFS=*)
            BOOT=zfs
            ;;
    esac
done
{% endhighlight %}

然后，就可以安装 ZFS 模块了。

{% highlight bash %}
# apt update
# apt install zfs-dkms zfs-initramfs
{% endhighlight %}

安装 grub，根据你的固件情况选择 grub-pc 或 grub-efi，我这里用的是 grub-pc。

{% highlight bash %}
# apt install grub-pc
{% endhighlight %}

先验证 grub 是否识别了 ZFS，如果没问题，就更新 `grub.cfg` 和 initramfs。

{% highlight bash %}
# ln -s /proc/self/mounts /etc/mtab
# grub-probe /
zfs
#如果输出以上信息，则进行后续操作
# update-grub
# update-initramfs -u -k all
{% endhighlight %}

然后将 grub 引导代码安装到 `/dev/sda`。

{% highlight bash %}
# grub-install /dev/sda
Installing for i386-pc platform.
Installation finished. No error reported.
{% endhighlight %}

必须确保安装成功，否则就无法引导了。

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

重新配置 locale。

{% highlight bash %}
# dpkg-reconfigure locales
{% endhighlight %}

然后卸载 `/mnt` 下挂载的所有文件系统，并且从 `/dev/sda` 引导。登录以后可以根据需要进行其他相关配置，本次安装完成。