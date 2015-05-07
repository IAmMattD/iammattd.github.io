---
title: 论 Linux 开发人员如何扯到自己蛋的
author: MattD
layout: post
permalink: /2015/05/07/how-linux-developers-fuck-themselves.html
categories: Linux
tags: [kernel, Linux]
---
在 Linux 的 4.1 版本内核中，开发人员又提交了一个无比蛋疼的修改：把所有 init_tss 符号都重命名为 cpu_tss 符号。肇事的就是[这次代码提交](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=24933b82c0d9a711475a5ef7904eb733f561e637)。

然后事情不对劲了，nvidia、AMD、Broadcom 的闭源驱动通通中枪了，为什么？因为这个符号是 GPL 的，也就是说，如果要引用这个符号，那么闭源驱动都有被迫开源的风险。

这次修改直接导致 nvidia 论坛和某个邮件列表上用户提出的问题好几天都没人解决，哀鸿遍野。内核的事，岂是凡夫俗子能随便指摘的？

<!-- more -->

然后呢，到了昨天，内核开发团队终于坐不住，自己打自己脸了，请围观[昨天的代码提交](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=de71ad2c97862eae1516aa36528cc3b317c17b2f)。

早知今日，你们何必当初呢？经测试，按照昨天的代码提交修改 4.1-rc2 源代码，并重新编译以后，nvidia 闭源驱动可以正常工作了。

看，Linux 开发人员就是这样扯到自己蛋的。
