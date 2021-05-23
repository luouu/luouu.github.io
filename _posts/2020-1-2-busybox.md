---
layout: post
title: BusyBox
categories: [rootfs]
description: 
keywords: 
topmost: false
---

* TOC
{:toc}

[下载地址](https://busybox.net/downloads/)

[https://salsa.debian.org/installer-team/busybox](https://salsa.debian.org/installer-team/busybox)

修改顶层Makefile，修改交叉编译器和芯片平台：

>CROSS_COMPILE = arm-none-linux-gnueabi- ARCH = arm

`make defconfig`，busybox提供了3种配置：defconfig (缺省配置)、allyesconfig（最大配置）、 allnoconfig（最小配置）。

`make menuconfig`

```conf
Busybox Settings  --->
    Build Options  --->
        [*]Build shared libbusybox
        [ ] Build BusyBox as a static binary (no shared libs) (NEW)

    (arm-anykav200-linux-uclibcgnueabi-) Cross compiler prefix 　//设置编译器
        
    Installation Options ("make install" behavior)  --->
        (../rootfs) Destination path for 'make install'   // 设置编译生成文件的存放路径
        What kind of applet links to install (as soft-links)  --->　//设置生成后的命令是指向busybox的软链接
  
  Busybox Library Tuning  --->
      (255) History size 
      [*]   History saving (NEW)   // 支持历史记录
      [*]   Tab completion (NEW)   // 支持Tab补全操作
```

`make; make install`

添加相应的库，用`readelf -d busybox`，查看依赖的库，将交叉编译环境下sysroot/lib目录下库拷贝到/lib。
