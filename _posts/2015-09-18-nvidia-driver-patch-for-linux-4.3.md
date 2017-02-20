---
title: 适用于 Linux 4.3 内核的 Nvidia 驱动补丁
author: MattD
layout: post
permalink: /2015/09/18/nvidia-driver-patch-for-linux-4.3.html
categories: Linux
tags: [kernel, nvidia]
---
上星期 Linux 4.3-rc1 发布以后，Nvidia 的闭源驱动又挂了，还好 Nvidia 论坛当天就有人提交了兼容性补丁，因此在这里记录一下。

补丁的用法就不说了，Gentoo 可以通过 user patch 功能打上去，其他发行版可以解包 Nvidia 的 .run 驱动，打上这个补丁，然后开始安装。

此补丁只在最新的 355.11 版驱动上测试通过，其他版本未测试。

<!-- more -->

以下是补丁正文：

{% highlight bash %}
--- a/kernel/nvidia/nv-procfs.c	2015-09-14 01:05:28.718354656 +0800
+++ b/kernel/nvidia/nv-procfs.c	2015-09-14 01:05:34.123373584 +0800
@@ -360,7 +360,8 @@
     registry_keys = ((nvl != NULL) ?
             nvl->registry_keys : nv_registry_keys);
 
-    return seq_printf(s, "Binary: \"%s\"\n", registry_keys);
+	seq_printf(s, "Binary: \"%s\"\n", registry_keys);
+	return 0;
 }
 
 static ssize_t
@@ -560,7 +561,8 @@
     void *v
 )
 {
-    return seq_puts(s, s->private);
+    seq_puts(s, s->private);
+	return 0;
 }
 
 NV_DEFINE_PROCFS_SINGLE_FILE(text_file);
{% endhighlight %}
