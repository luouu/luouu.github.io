---
layout: post
title: linux内核配置与编译
categories: [kernel]
description: 
keywords: 
topmost: false
---

* TOC
{:toc}

## 配置

```shell
CROSS_COMPILE=arm-none-linux-gnueabi- ARCH=arm make menuconfig
```

## 编译

```shell
make
```

* 屏蔽编译信息

>make > /dev/null

* 使用`make -j<n>`参数加速编译过程

* 指定编译某些模块

>make M=drivers/i2c/ modules

* 使用 verbose 模式，将每一步执行的命令都打印出来，并重定向到一个文件中去，这样以后可以方便地查找模块之间的依赖关系。

>make V=1 > ~/bak.txt

* 使用 ccache 提高编译速度，使用 ccache时，需要更改源码树根目录下面的 Makefile 文件，在 CC 和 HOSTCC 变量的定义前添加 ccache。[ccache主页](http://ccache.samba.org/)。

```makefile
CC = ccache $(CROSS_COMPILE)gcc
HOSTCC = ccache gcc
```

### 编译文档

使用下面的一些命令可以在 Documentation/DocBook/目录下，生成一些文档。

```shell
make htmldocs //生成 HTML 文件
make pdfdocs  //生成 PDF 文件
make psdocs   //生成 Postscript 文件
make mandocs  //生成 Kernel API 手册
make installmandocs //将 Kernel API 手册页安装到 man 程序能够找到的目录中
```

## 常见的错误

```x
make menuconfig requires the ncurses libraries.
```

sudo apt-get install libncurses5-dev

```x
mkimage　command not found – U-Boot images will not be built
```

sudo apt-get install u-boot-tools

```x
module license 'unspecified' taints kernel.
Disabling lock debugging due to kernel taint
```

由于多个C文件中的一个xxx.o文件和模块目标文件xxx.o重名。