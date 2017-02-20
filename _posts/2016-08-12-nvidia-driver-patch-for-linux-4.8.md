---
title: 适用于 Linux 4.8 内核的 Nvidia 驱动补丁
author: MattD
layout: post
permalink: /2016/08/12/nvidia-driver-patch-for-linux-4.8.html
categories: Linux
tags: [kernel, nvidia]
---

Linux 内核于本周发布了 4.8 的第一个 rc 版。

除了 4.7 内核中 `radix_tree` 函数的问题，API 也有其他变化，所以导致 Nvidia 驱动又得改一下 patch 才能顺利编译安装。

内核的 patch 依然不变，需要把 `radix_tree` 函数重命名。

<!-- more -->

{% highlight bash %}
--- a/include/linux/radix-tree.h	2016-05-31 21:12:31.143579722 +0800
+++ b/include/linux/radix-tree.h	2016-05-31 21:13:09.593732659 +0800
@@ -124,7 +124,7 @@
 	(root)->rnode = NULL;						\
 } while (0)
 
-static inline bool radix_tree_empty(struct radix_tree_root *root)
+static inline bool radix_tree_is_empty(struct radix_tree_root *root)
 {
 	return root->rnode == NULL;
 }
--- a/kernel/irq/irqdomain.c	2016-05-31 21:14:30.456053855 +0800
+++ b/kernel/irq/irqdomain.c	2016-05-31 21:15:01.909178644 +0800
@@ -139,7 +139,7 @@
 {
 	mutex_lock(&irq_domain_mutex);
 
-	WARN_ON(!radix_tree_empty(&domain->revmap_tree));
+	WARN_ON(!radix_tree_is_empty(&domain->revmap_tree));
 
 	list_del(&domain->link);
{% endhighlight %}

然后是针对 Nvidia 驱动的 patch：

{% highlight bash %}
--- a/kernel/nvidia-drm/nvidia-drm-drv.c	2016-07-25 13:23:10.000000000 +0200
+++ b/kernel/nvidia-drm/nvidia-drm-drv.c	2016-08-02 17:09:53.500398878 +0200
@@ -36,6 +36,7 @@
 #include "nvidia-drm-ioctl.h"
 
 #include <drm/drmP.h>
+#include <drm/drm_auth.h>
 
 #include <drm/drm_crtc_helper.h>
 
@@ -419,7 +420,7 @@
 
 static
 void nvidia_drm_master_drop(struct drm_device *dev,
-                            struct drm_file *file_priv, bool from_release)
+                            struct drm_file *file_priv)
 {
     struct nvidia_drm_device *nv_dev = dev->dev_private;
     int ret;
@@ -452,7 +453,7 @@
     mutex_lock(&dev->master_mutex);
 
     if (!file_priv->is_master ||
-        !file_priv->minor->master)
+        !file_priv->master)
     {
         goto done;
     }
@@ -473,7 +474,7 @@
      * NVKMS modeset ownership, because nvidia_drm_master_set()'s call to
      * grabOwnership() will fail.
      */
-    drm_master_put(&file_priv->minor->master);
+    drm_master_put(&file_priv->master);
     file_priv->is_master = 0;
 
     ret = 0;

--- a/kernel/nvidia-drm/nvidia-drm-fb.c	2016-07-25 13:23:10.000000000 +0200
+++ b/kernel/nvidia-drm/nvidia-drm-fb.c	2016-07-31 05:37:57.013950532 +0200
@@ -114,7 +114,7 @@
      * We don't support any planar format, pick up first buffer only.
      */
 
-    gem = drm_gem_object_lookup(dev, file, cmd->handles[0]);
+    gem = drm_gem_object_lookup(file, cmd->handles[0]);
 
     if (gem == NULL)
     {

--- a/kernel/nvidia-drm/nvidia-drm-gem.c	2016-07-25 13:23:10.000000000 +0200
+++ b/kernel/nvidia-drm/nvidia-drm-gem.c	2016-07-31 05:37:57.013950532 +0200
@@ -408,7 +408,7 @@
 
     mutex_lock(&dev->struct_mutex);
 
-    gem = drm_gem_object_lookup(dev, file, handle);
+    gem = drm_gem_object_lookup(file, handle);
 
     if (gem == NULL)
     {

--- a/kernel/nvidia-drm/nvidia-drm-modeset.c	2016-07-25 13:23:10.000000000 +0200
+++ b/kernel/nvidia-drm/nvidia-drm-modeset.c	2016-08-02 17:14:57.895422720 +0200
@@ -675,7 +675,7 @@
         goto failed;
     }
 
-    drm_atomic_helper_swap_state(dev, state);
+    drm_atomic_helper_swap_state(state, true);
 
     nvidia_drm_update_head_mode_config(state, requested_config);
 
{% endhighlight %}

`radix_tree` 函数的问题看来是无法指望内核上游去修复了，Nvidia 那边似乎也不积极，暂时先这么凑合着吧。
