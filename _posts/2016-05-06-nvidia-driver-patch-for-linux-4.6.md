---
title: 适用于 Linux 4.6 内核的 Nvidia 驱动补丁
author: MattD
layout: post
permalink: /2016/05/06/nvidia-driver-patch-for-linux-4.6.html
categories: Linux
tags: [kernel, nvidia]
---
经过几个 RC 版以后，4.6 内核的特性和代码算是稳定下来了，相应的 Nvidia 驱动也是几经更新。

目前这个补丁适用于 4.6-rc3 以后的所有内核版本，针对的 Nvidia 驱动为 36x 版本，35x 版本的驱动未经测试。

相信等下周或下下周 4.6 内核正式发布以后，也不需要再改了。

<!-- more -->

以下是补丁正文：

{% highlight bash %}
--- a/kernel/nvidia-drm/nvidia-drm-fb.c
+++ b/kernel/nvidia-drm/nvidia-drm-fb.c
@@ -77,7 +77,7 @@
 static struct drm_framebuffer *internal_framebuffer_create
 (
     struct drm_device *dev,
-    struct drm_file *file, struct drm_mode_fb_cmd2 *cmd,
+    struct drm_file *file, const struct drm_mode_fb_cmd2 *cmd,
     uint64_t nvkms_params_ptr,
     uint64_t nvkms_params_size
 )
@@ -199,7 +199,7 @@
 struct drm_framebuffer *nvidia_drm_framebuffer_create
 (
     struct drm_device *dev,
-    struct drm_file *file, struct drm_mode_fb_cmd2 *cmd
+    struct drm_file *file, const struct drm_mode_fb_cmd2 *cmd
 )
 {
     return internal_framebuffer_create(dev, file, cmd, 0, 0);

--- a/kernel/nvidia-drm/nvidia-drm-fb.h
+++ b/kernel/nvidia-drm/nvidia-drm-fb.h
@@ -45,7 +45,7 @@
 struct drm_framebuffer *nvidia_drm_framebuffer_create
 (
     struct drm_device *dev,
-    struct drm_file *file, struct drm_mode_fb_cmd2 *cmd
+    struct drm_file *file, const struct drm_mode_fb_cmd2 *cmd
 );
 
 int nvidia_drm_add_nvkms_fb(

--- a/kernel/nvidia-drm/nvidia-drm-linux.c
+++ b/kernel/nvidia-drm/nvidia-drm-linux.c
@@ -31,6 +31,7 @@
 
 #if defined(NV_DRM_AVAILABLE)
 
+#include "nv-mm.h"
 #include "nv-pgprot.h"
 
 MODULE_PARM_DESC(
@@ -121,8 +122,7 @@
 
     down_read(&mm->mmap_sem);
 
-    pages_pinned = get_user_pages(current, mm,
-                                  address, pages_count, write, force,
+    pages_pinned = NV_GET_USER_PAGES(address, pages_count, write, force,
                                   user_pages, NULL);
     up_read(&mm->mmap_sem);
{% endhighlight %}
