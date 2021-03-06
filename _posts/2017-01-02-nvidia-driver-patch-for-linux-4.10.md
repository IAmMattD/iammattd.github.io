---
title: 适用于 Linux 4.10 内核的 Nvidia 驱动补丁
author: MattD
layout: post
permalink: /2017/01/02/nvidia-driver-patch-for-linux-4.8.html
categories: Linux
tags: [kernel, nvidia]
---

4.10 内核于上周发布了第一个 rc 版。

除了大量的驱动更新，当然喜闻乐见地又把 Nvidia 闭源驱动搞挂了。

虽然 Nvidia 的开发论坛上早有人发布了 for 4.10 内核的补丁，但是那个补丁似乎只适用于 pre-rc，也就是 rc0 版本。

而 rc1 的发布以后又修改了 `CPU Hotplug` 相关的几个函数，因此那个补丁并不适用。

为了编译通过，上周我只是简单粗暴地删除了 Nvidia 驱动文件中跟被删除的 `CPU Hotplug` 函数相关的一些代码。但是这样的做法实在是过于 dirty。

<!-- more -->

好在，这星期 Nvidia 开发论坛上终于放出了适用于 4.10 内核的正式补丁。

跟以往一样，只在最新的 370.26 驱动上通过编译测试，其他版本不保证。

{% highlight bash %}
--- a/kernel/common/inc/nv-linux.h	2016-12-09 02:17:49.000000000 +0100
+++ b/kernel/common/inc/nv-linux.h	2016-12-31 10:55:17.637374373 +0100
@@ -294,7 +294,8 @@
 
 extern int nv_pat_mode;
 
-#if defined(CONFIG_HOTPLUG_CPU)
+//#if defined(CONFIG_HOTPLUG_CPU)
+#if 0
 #define NV_ENABLE_HOTPLUG_CPU
 #include <linux/cpu.h>              /* CPU hotplug support              */
 #include <linux/notifier.h>         /* struct notifier_block, etc       */

--- a/kernel/common/inc/nv-mm.h	2016-12-09 02:17:49.000000000 +0100
+++ b/kernel/common/inc/nv-mm.h	2016-12-15 11:51:32.638165272 +0100
@@ -83,7 +83,7 @@
             if (force)
                 flags |= FOLL_FORCE;
 
-            return get_user_pages_remote(tsk, mm, start, nr_pages, flags, pages, vmas);
+            return get_user_pages_remote(tsk, mm, start, nr_pages, flags, pages, vmas, NULL);
         }
     #endif
 #else
Only in NVIDIA-Linux-x86_64-375.26.patched/kernel: nv_compiler.h

--- a/kernel/nvidia/nv-p2p.c	2016-12-09 02:17:49.000000000 +0100
+++ b/kernel/nvidia/nv-p2p.c	2016-12-15 11:51:32.638165272 +0100
@@ -146,7 +146,7 @@
 int nvidia_p2p_get_pages(
     uint64_t p2p_token,
     uint32_t va_space,
-    uint64_t virtual_address,
+    uint64_t address,
     uint64_t length,
     struct nvidia_p2p_page_table **page_table,
     void (*free_callback)(void * data),
@@ -211,7 +211,7 @@
     }
 
     status = rm_p2p_get_pages(sp, p2p_token, va_space,
-            virtual_address, length, physical_addresses, wreqmb_h,
+            address, length, physical_addresses, wreqmb_h,
             rreqmb_h, &entries, &gpu_uuid, *page_table,
             free_callback, data);
     if (status != NV_OK)
@@ -286,7 +286,7 @@
 
     if (bGetPages)
     {
-        rm_p2p_put_pages(sp, p2p_token, va_space, virtual_address,
+        rm_p2p_put_pages(sp, p2p_token, va_space, address,
                 gpu_uuid, *page_table);
     }
 
@@ -329,7 +329,7 @@
 int nvidia_p2p_put_pages(
     uint64_t p2p_token,
     uint32_t va_space,
-    uint64_t virtual_address,
+    uint64_t address,
     struct nvidia_p2p_page_table *page_table
 )
 {
@@ -343,7 +343,7 @@
         return rc;
     }
 
-    status = rm_p2p_put_pages(sp, p2p_token, va_space, virtual_address,
+    status = rm_p2p_put_pages(sp, p2p_token, va_space, address,
             page_table->gpu_uuid, page_table);
     if (status == NV_OK)
         nvidia_p2p_free_page_table(page_table);

--- a/kernel/nvidia/nv-pat.c	2016-12-09 02:17:49.000000000 +0100
+++ b/kernel/nvidia/nv-pat.c	2016-12-31 10:45:47.380405916 +0100
@@ -217,7 +217,7 @@
             else
                 NV_SMP_CALL_FUNCTION(nv_setup_pat_entries, hcpu, 1);
             break;
-        case CPU_DOWN_PREPARE:
+        case CPU_DOWN_PREPARE_FROZEN:
             if (cpu == (NvUPtr)hcpu)
                 nv_restore_pat_entries(NULL);
             else

--- a/kernel/nvidia-drm/nvidia-drm-fence.c	2016-12-09 02:13:39.000000000 +0100
+++ b/kernel/nvidia-drm/nvidia-drm-fence.c	2016-12-15 11:51:32.638165272 +0100
@@ -31,7 +31,7 @@
 
 #if defined(NV_DRM_DRIVER_HAS_GEM_PRIME_RES_OBJ)
 struct nv_fence {
-    struct fence base;
+    struct dma_fence base;
     spinlock_t lock;
 
     struct nvidia_drm_device *nv_dev;
@@ -51,7 +51,7 @@
 
 static const char *nvidia_drm_gem_prime_fence_op_get_driver_name
 (
-    struct fence *fence
+    struct dma_fence *fence
 )
 {
     return "NVIDIA";
@@ -59,7 +59,7 @@
 
 static const char *nvidia_drm_gem_prime_fence_op_get_timeline_name
 (
-    struct fence *fence
+    struct dma_fence *fence
 )
 {
     return "nvidia.prime";
@@ -67,7 +67,7 @@
 
 static bool nvidia_drm_gem_prime_fence_op_signaled
 (
-    struct fence *fence
+    struct dma_fence *fence
 )
 {
     struct nv_fence *nv_fence = container_of(fence, struct nv_fence, base);
@@ -99,7 +99,7 @@
 
 static bool nvidia_drm_gem_prime_fence_op_enable_signaling
 (
-    struct fence *fence
+    struct dma_fence *fence
 )
 {
     bool ret = true;
@@ -107,7 +107,7 @@
     struct nvidia_drm_gem_object *nv_gem = nv_fence->nv_gem;
     struct nvidia_drm_device *nv_dev = nv_fence->nv_dev;
 
-    if (fence_is_signaled(fence))
+    if (dma_fence_is_signaled(fence))
     {
         return false;
     }
@@ -132,7 +132,7 @@
     }
 
     nv_gem->fenceContext.softFence = fence;
-    fence_get(fence);
+    dma_fence_get(fence);
 
 unlock_struct_mutex:
     mutex_unlock(&nv_dev->dev->struct_mutex);
@@ -142,7 +142,7 @@
 
 static void nvidia_drm_gem_prime_fence_op_release
 (
-    struct fence *fence
+    struct dma_fence *fence
 )
 {
     struct nv_fence *nv_fence = container_of(fence, struct nv_fence, base);
@@ -151,7 +151,7 @@
 
 static signed long nvidia_drm_gem_prime_fence_op_wait
 (
-    struct fence *fence,
+    struct dma_fence *fence,
     bool intr,
     signed long timeout
 )
@@ -166,12 +166,12 @@
      * that it should never get hit during normal operation, but not so long
      * that the system becomes unresponsive.
      */
-    return fence_default_wait(fence, intr,
+    return dma_fence_default_wait(fence, intr,
                               (timeout == MAX_SCHEDULE_TIMEOUT) ?
                                   msecs_to_jiffies(96) : timeout);
 }
 
-static const struct fence_ops nvidia_drm_gem_prime_fence_ops = {
+static const struct dma_fence_ops nvidia_drm_gem_prime_fence_ops = {
     .get_driver_name = nvidia_drm_gem_prime_fence_op_get_driver_name,
     .get_timeline_name = nvidia_drm_gem_prime_fence_op_get_timeline_name,
     .signaled = nvidia_drm_gem_prime_fence_op_signaled,
@@ -281,7 +281,7 @@
     bool force
 )
 {
-    struct fence *fence = nv_gem->fenceContext.softFence;
+    struct dma_fence *fence = nv_gem->fenceContext.softFence;
 
     WARN_ON(!mutex_is_locked(&nv_dev->dev->struct_mutex));
 
@@ -297,10 +297,10 @@
 
         if (force || nv_fence_ready_to_signal(nv_fence))
         {
-            fence_signal(&nv_fence->base);
+            dma_fence_signal(&nv_fence->base);
 
             nv_gem->fenceContext.softFence = NULL;
-            fence_put(&nv_fence->base);
+            dma_fence_put(&nv_fence->base);
 
             nvKms->disableChannelEvent(nv_dev->pDevice,
                                        nv_gem->fenceContext.cb);
@@ -316,7 +316,7 @@
 
         nv_fence = container_of(fence, struct nv_fence, base);
 
-        fence_signal(&nv_fence->base);
+        dma_fence_signal(&nv_fence->base);
     }
 }
 
@@ -509,7 +509,7 @@
      * fence_context_alloc() cannot fail, so we do not need to check a return
      * value.
      */
-    nv_gem->fenceContext.context = fence_context_alloc(1);
+    nv_gem->fenceContext.context = dma_fence_context_alloc(1);
 
     ret = nvidia_drm_gem_prime_fence_import_semaphore(
               nv_dev, nv_gem, p->index,
@@ -666,13 +666,13 @@
     nv_fence->nv_gem = nv_gem;
 
     spin_lock_init(&nv_fence->lock);
-    fence_init(&nv_fence->base, &nvidia_drm_gem_prime_fence_ops,
+    dma_fence_init(&nv_fence->base, &nvidia_drm_gem_prime_fence_ops,
                &nv_fence->lock, nv_gem->fenceContext.context,
                p->sem_thresh);
 
     reservation_object_add_excl_fence(&nv_gem->fenceContext.resv,
                                       &nv_fence->base);
-    fence_put(&nv_fence->base); /* Reservation object has reference */
+    dma_fence_put(&nv_fence->base); /* Reservation object has reference */
 
     ret = 0;

--- a/kernel/nvidia-drm/nvidia-drm-gem.h	2016-12-09 02:13:39.000000000 +0100
+++ b/kernel/nvidia-drm/nvidia-drm-gem.h	2016-12-15 11:51:32.638165272 +0100
@@ -98,7 +98,7 @@
         /* Software signaling structures */
         struct NvKmsKapiChannelEvent *cb;
         struct nvidia_drm_gem_prime_soft_fence_event_args *cbArgs;
-        struct fence *softFence; /* Fence for software signaling */
+        struct dma_fence *softFence; /* Fence for software signaling */
     } fenceContext;
 #endif
 };

--- a/kernel/nvidia-drm/nvidia-drm-modeset.c	2016-12-09 02:13:39.000000000 +0100
+++ b/kernel/nvidia-drm/nvidia-drm-modeset.c	2016-12-15 11:51:32.638165272 +0100
@@ -69,8 +69,7 @@
 
 void nvidia_drm_atomic_state_free(struct drm_atomic_state *state)
 {
-    struct nvidia_drm_atomic_state *nv_state =
-                    to_nv_atomic_state(state);
+    struct nvidia_drm_atomic_state *nv_state = to_nv_atomic_state(state);
     drm_atomic_state_default_release(state);
     nvidia_drm_free(nv_state);
 }
@@ -645,7 +644,7 @@
 
     wake_up_all(&nv_dev->pending_commit_queue);
 
-    drm_atomic_state_free(state);
+    drm_atomic_state_put(state);
 
 #if !defined(NV_DRM_MODE_CONFIG_FUNCS_HAS_ATOMIC_STATE_ALLOC)
     nvidia_drm_free(requested_config);
@@ -983,7 +982,7 @@
      * drm_atomic_commit().
      */
     if (ret != 0) {
-        drm_atomic_state_free(state);
+        drm_atomic_state_put(state);
     }
 
     drm_modeset_unlock_all(dev);

--- a/kernel/nvidia-drm/nvidia-drm-priv.h	2016-12-09 02:13:39.000000000 +0100
+++ b/kernel/nvidia-drm/nvidia-drm-priv.h	2016-12-15 11:51:32.638165272 +0100
@@ -34,7 +34,7 @@
 #endif
 
 #if defined(NV_DRM_DRIVER_HAS_GEM_PRIME_RES_OBJ)
-#include <linux/fence.h>
+#include <linux/dma-fence.h>
 #include <linux/reservation.h>
 #endif
 
--- a/kernel/nvidia-uvm/uvm8.c	2016-12-09 02:17:46.000000000 +0100
+++ b/kernel/nvidia-uvm/uvm8.c	2016-12-15 11:51:32.638165272 +0100
@@ -101,7 +101,7 @@
 // so we force it to fail instead.
 static int uvm_vm_fault_sigbus(struct vm_area_struct *vma, struct vm_fault *vmf)
 {
-    UVM_DBG_PRINT_RL("Fault to address 0x%p in disabled vma\n", vmf->virtual_address);
+    UVM_DBG_PRINT_RL("Fault to address 0x%p in disabled vma\n", vmf->address);
     vmf->page = NULL;
     return VM_FAULT_SIGBUS;
 }
@@ -315,7 +315,7 @@
 {
     uvm_va_space_t *va_space = uvm_va_space_get(vma->vm_file);
     uvm_va_block_t *va_block;
-    NvU64 fault_addr = (NvU64)(uintptr_t)vmf->virtual_address;
+    NvU64 fault_addr = (NvU64)(uintptr_t)vmf->address;
     bool is_write = vmf->flags & FAULT_FLAG_WRITE;
     NV_STATUS status = uvm_global_get_status();
     bool tools_enabled;

--- a/kernel/nvidia-uvm/uvm8_test.c	2016-12-09 02:17:46.000000000 +0100
+++ b/kernel/nvidia-uvm/uvm8_test.c	2016-12-15 11:51:32.639165272 +0100
@@ -103,7 +103,7 @@
     return NV_ERR_INVALID_STATE;
 }
 
-static NV_STATUS uvm8_test_get_kernel_virtual_address(
+static NV_STATUS uvm8_test_get_kernel_address(
         UVM_TEST_GET_KERNEL_VIRTUAL_ADDRESS_PARAMS *params,
         struct file *filp)
 {
@@ -173,7 +173,7 @@
         UVM_ROUTE_CMD_STACK(UVM_TEST_RANGE_GROUP_RANGE_COUNT,       uvm8_test_range_group_range_count);
         UVM_ROUTE_CMD_STACK(UVM_TEST_GET_PREFETCH_FAULTS_REENABLE_LAPSE, uvm8_test_get_prefetch_faults_reenable_lapse);
         UVM_ROUTE_CMD_STACK(UVM_TEST_SET_PREFETCH_FAULTS_REENABLE_LAPSE, uvm8_test_set_prefetch_faults_reenable_lapse);
-        UVM_ROUTE_CMD_STACK(UVM_TEST_GET_KERNEL_VIRTUAL_ADDRESS,    uvm8_test_get_kernel_virtual_address);
+        UVM_ROUTE_CMD_STACK(UVM_TEST_GET_KERNEL_VIRTUAL_ADDRESS,    uvm8_test_get_kernel_address);
         UVM_ROUTE_CMD_STACK(UVM_TEST_PMA_ALLOC_FREE,                uvm8_test_pma_alloc_free);
         UVM_ROUTE_CMD_STACK(UVM_TEST_PMM_ALLOC_FREE_ROOT,           uvm8_test_pmm_alloc_free_root);
         UVM_ROUTE_CMD_STACK(UVM_TEST_PMM_INJECT_PMA_EVICT_ERROR,    uvm8_test_pmm_inject_pma_evict_error);

--- a/kernel/nvidia-uvm/uvm_lite.c	2016-12-09 02:17:46.000000000 +0100
+++ b/kernel/nvidia-uvm/uvm_lite.c	2016-12-15 11:51:32.639165272 +0100
@@ -1333,7 +1333,7 @@
 #if defined(NV_VM_OPERATIONS_STRUCT_HAS_FAULT)
 int _fault(struct vm_area_struct *vma, struct vm_fault *vmf)
 {
-    unsigned long vaddr = (unsigned long)vmf->virtual_address;
+    unsigned long vaddr = (unsigned long)vmf->address;
     struct page *page = NULL;
     int retval;
{% endhighlight %}

目前在 4.10-rc2 内核上也测试通过。
