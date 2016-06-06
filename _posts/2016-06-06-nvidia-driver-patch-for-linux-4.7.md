---
title: 适用于 Linux 4.7 内核的 Nvidia 驱动补丁
author: MattD
layout: post
permalink: /2016/06/06/nvidia-driver-patch-for-linux-4.7.html
categories: Linux
tags: [kernel, nvidia]
---

Linux 内核于上周发布了 4.7 的第一个 rc 版，果不其然，闭源的 Nvidia 驱动又挂掉了。而且，这次依然是内核的锅。

具体来说，Linux 内核在 `/include/linux/radix-tree.h` 里面引入了一个新的函数，名为 `radix_tree_empty`。然而很不巧，这个函数和 Nvidia 闭源驱动里面的一个现有函数同名，因此就会导致 Nvidia 驱动在编译时候产生混乱。

除此之外，4.7 内核还删除了一个调用参数，而 Nvidia 闭源驱动还没有相应更新。

因此，为了正常编译 Nvidia 驱动，需要对内核源代码和 Nvidia 驱动文件同时做一些修改。

<!-- more -->

首先是针对内核的一个 patch，用于重命名那个同名函数：

{% highlight bash %}
--- include/linux/radix-tree.h.orig	2016-05-31 21:12:31.143579722 +0800
+++ include/linux/radix-tree.h	2016-05-31 21:13:09.593732659 +0800
@@ -124,7 +124,7 @@
 	(root)->rnode = NULL;						\
 } while (0)
 
-static inline bool radix_tree_empty(struct radix_tree_root *root)
+static inline bool radix_tree_is_empty(struct radix_tree_root *root)
 {
 	return root->rnode == NULL;
 }
--- kernel/irq/irqdomain.c.orig	2016-05-31 21:14:30.456053855 +0800
+++ kernel/irq/irqdomain.c	2016-05-31 21:15:01.909178644 +0800
@@ -139,7 +139,7 @@
 {
 	mutex_lock(&irq_domain_mutex);
 
-	WARN_ON(!radix_tree_empty(&domain->revmap_tree));
+	WARN_ON(!radix_tree_is_empty(&domain->revmap_tree));
 
 	list_del(&domain->link);
{% endhighlight %}

然后是针对 Nvidia 驱动的 patch：

{% highlight bash %}
--- a/kernel/nvidia-drm/nvidia-drm-fb.c	2016-06-06 19:57:00.370515382 +0800
+++ b/kernel/nvidia-drm/nvidia-drm-fb.c	2016-06-06 19:57:34.728704809 +0800
@@ -114,7 +114,7 @@
      * We don't support any planar format, pick up first buffer only.
      */
 
-    gem = drm_gem_object_lookup(dev, file, cmd->handles[0]);
+    gem = drm_gem_object_lookup(file, cmd->handles[0]);
 
     if (gem == NULL)
     {
--- a/kernel/nvidia-drm/nvidia-drm-gem.c	2016-06-06 19:58:00.225844750 +0800
+++ b/kernel/nvidia-drm/nvidia-drm-gem.c	2016-06-06 19:58:28.783000882 +0800
@@ -408,7 +408,7 @@
 
     mutex_lock(&dev->struct_mutex);
 
-    gem = drm_gem_object_lookup(dev, file, handle);
+    gem = drm_gem_object_lookup(file, handle);
 
     if (gem == NULL)
     {
{% endhighlight %}

不过虽然 Nvidia 驱动模块可以顺利通过编译，但是用新内核启动之后，控制台会抛出非常多的 traceback。

所以这只是权宜之计，最好还是等内核上游解决那个同名函数的问题。
