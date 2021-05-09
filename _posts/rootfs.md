<!--
 * @Description: rootfs
 * @Author: luo_u
 * @Date: 2020-05-18 11:49:24
 * @LastEditTime: 2020-10-26 10:57:31
-->

[TOC]


## BusyBox

[下载地址](https://busybox.net/downloads/)

https://salsa.debian.org/installer-team/busybox

1.修改顶层Makefile，修改交叉编译器和硬件平台：
>CROSS_COMPILE = arm-none-linux-gnueabi-

>ARCH = arm

2. `make defconfig`，busybox提供了3种配置：defconfig (缺省配置)、allyesconfig（最大配置）、 allnoconfig（最小配置）。

3. `make menuconfig` 配置

```
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

`make`; `make install`

4.　添加相应的库
用`readelf -d busybox`，查看依赖的库，将交叉编译环境下sysroot/lib目录下库拷贝到/lib。



## flash文件系统
flash芯片可以被划分为多个分区，各分区可以采用不同的文件系统。flash文件系统是基于MTD驱动层的，MTD(Memory Technology Device,存储技术设备)为底层硬件(闪存)和上层(文件系统)之间提供一个统一的抽象接口，专门针对各种非易失性存储器设计，因而对Flash有更好的支持、管理和基于扇区的擦除、读/写操作接口。


### jffs
JFFS文件系统最早是由瑞典Axis Communications公司基于Linux2.0的内核为嵌入式系统开发的文件系统。JFFS2是RedHat公司基于JFFS开发的闪存文件系 统，最初是针对RedHat公司的嵌入式产品eCos开发的嵌入式文件系统，所以JFFS2也可以用在Linux, uCLinux中。

Jffs2: 日志闪存文件系统版本2 (Journalling Flash FileSystem v2)

主要用于NOR型闪存，基于MTD驱动层，特点是：**可读写**的、支持数据压缩的、基于哈希表的日志型文件系统，并提供了崩溃/掉电安全保护，提供“写平衡”支持等。缺点主要是当文件系统已满或接近满时，因为垃圾收集的关系而使jffs2的运行速度大大放慢。

目前jffs3正在开发中。关于jffs系列文件系统的使用详细文档，可参考MTD补丁包中mtd-jffs-HOWTO.txt。

*jffsx不适合用于NAND闪存*主要是因为NAND闪存的容量一般较大，这样导致jffs为维护日志节点所占用的内存空间迅速增大，另 外，jffsx文件系统在挂载时需要扫描整个FLASH的内容，以找出所有的日志节点，建立文件结构，对于大容量的NAND闪存会耗费大量时间。


####　mkfs.jffs2
直接安装`mkfs.jffs2`工具：sudo apt-get install mtd-utils，或编译源码。
```shell
sudo apt-get install liblzo2-2 libuuid1 zlib1g liblzo2-dev uuid-dev libacl1-dev  zlib1g-dev
./configure --prefix=/home/luo_u/usr
make 
make install
```

*使用说明*:
```shell
./mkfs.jffs2 -v -d rootfs/ -l -s 256 -e 4096 -m none -o root.jffs2

-p, --pad[=SIZE]        使用0xff填充文件系统到指定大小，不指定则只填充完最后一个擦除块
-r, -d, --root=DIR      使用指定的目录内容构建文件系统
-s, --pagesize=SIZE     使用指定的页大小（最大数据节点大小） (default: 4KiB)
-e, --eraseblock=SIZE   指定擦除块的大小 (default: 64KiB)
-c, --cleanmarker=SIZE  擦除标记的大小 (default 12)
-m, --compr-mode=MODE   选择压缩模式(default: priortiry)
-x, --disable-compressor=COMPRESSOR_NAME	禁用指定的压缩算法
-X, --enable-compressor=COMPRESSOR_NAME	启用指定的压缩算法
-y, --compressor-priority=PRIORITY:COMPRESSOR_NAME	设置压缩算法的优先级
-L, --list-compressors  列出可用的压缩算法
-t, --test-compression  测试压缩算法
-n, --no-cleanmarkers   不添加擦除标记到擦除块
-o, --output=FILE       指定输出镜像文件名称
-l, --little-endian     创建一个小端的文件系统
-b, --big-endian        创建一个大端的文件系统
-q, --squash            压缩权限和设置所有文件的拥有者为root
-U, --squash-uids       设置所有文件的拥有者为root
-P, --squash-perms      压缩所有文件的权限
-v  可视操作
```

*挂载分区*:
```shell
mount -t jffs2 /dev/mtdblock1 /mnt
```


### yaffs
yaffs/yaffs2(Yet Another Flash File System)是专为嵌入式系统使用NAND型闪存而设计的一种日志型文件系统。与jffs2相比，它减少了一些功能(例如不支持数 据压缩)，所以速度更快，挂载时间很短，对内存的占用较小。另外，它还是跨平台的文件系统，除了Linux和eCos，还支持WinCE, pSOS和ThreadX等。

yaffs/yaffs2自带NAND芯片的驱动，并且为嵌入式系统提供了直接访问文件系统的API，用户可以不使用Linux中的MTD与VFS，直接对文件系统操作。当然，yaffs也可与MTD驱动程序配合使用。

yaffs与yaffs2的主要区别在于，前者仅支持小页(512 Bytes) NAND闪存，后者则可支持大页(2KB) NAND闪存。同时，yaffs2在内存空间占用、垃圾回收速度、读/写速度等方面均有大幅提升。

https://yaffs.net/documents/how-yaffs-works


### cramfs
Cramfs(Compressed ROM File System)是Linux的创始人 Linus Torvalds参与开发的一种只读的压缩文件系统。它也基于MTD驱动程序。

在cramfs文件系统中，每一页(4KB)被单独压缩，可以随机页访问，其压缩比高达2:1,为嵌入式系统节省大量的Flash存储空间，使系统可通过更低容量的FLASH存储相同的文件，从而降低系统成本。

Cramfs文件系统以压缩方式存储，在运行时解压缩，所以不支持应用程序以XIP方式运行，所有的应用程序要求被拷到RAM里去运行，但这并 不代表比Ramfs需求的RAM空间要大一点，因为Cramfs是采用分页压缩的方式存放档案，在读取档案时，不会一下子就耗用过多的内存空间，只针对目 前实际读取的部分分配内存，尚没有读取的部分不分配内存空间，当我们读取的档案不在内存时，Cramfs文件系统自动计算压缩后的资料所存的位置，再即时 解压缩到RAM中。

另外，它的速度快，效率高，其只读的特点有利于保护文件系统免受破坏，提高了系统的可靠性。由于以上特性，Cramfs在嵌入式系统中应用广泛。但是它的**只读**属性同时又是它的一大缺陷，使得用户无法对其内容对进扩充。Cramfs映像通常是放在Flash中，但是也能放在别的文件系统里，使用loopback 设备可以把它安装别的文件系统里。

单个文件大小不能超过16MB、文件系统大小略大于256MB（最后一个文件允许超过256MB空间范围，即文件系统总大小不超过272MB）。CramFS的gid只保存8位，mkcramfs会简单的将gid截断保留最后8位。CramFS支持硬链接，但是被硬链接的文件引用计数不会增加。CramFS文件没有时间戳，所有文件的创建/访问时间戳都是1970年1月1日 0:00:00 GMT。CramFS的镜像只支持被同样字节对齐方式的机器创建和挂载使用，页面大小只支持4KB。



### romfs
传统型的Romfs文件系统是一种简单的、紧凑的、只读的文件系统，不支持动态擦写保存，按顺序存放数据，因而支持应用程序以 XIP(eXecute In Place，片内运行)方式运行，在系统运行时，节省RAM空间。uClinux系统通常采用Romfs文件系统。
　　
其他文件系统：fat/fat32也可用于实际嵌入式系统的扩展存储器(例如PDA, Smartphone, 数码相机等的SD卡)，这主要是为了更好的与最流行的Windows桌面操作系统相兼容。ext2也可以作为嵌入式Linux的文件系统，不过将它用于 FLASH闪存会有诸多弊端。
　　

### squashfs
SquashFS [1]  是一套基于Linux内核使用的压缩**只读**文件系统。该文件系统能够压缩系统内的文档,inode以及目录，文件最大支持2^64字节。
-　Squashfs 4.2 : 最新的版本，并适用于2.6.29版本以后的Linux内核。
-　Squashfs 4.1 : 支持XZ压缩，并适用于2.6.29版本以后的Linux内核。
-　Squashfs 3.4 : 该版本是3.X的最后一个版本，并适用于2.6.29版本之前的内核。

安装`mksquashfs`工具：sudo apt-get install squashfs-tools

*使用说明*:
```shell
mksquashfs rootfs/usr usr.sqsh4 -noappend -comp xz
```

*挂载分区*:
```shell
mount -t squashfs /dev/mtdblock6 /usr
```


### RAM的文件系统

#### ramdisk
Ramdisk是将一部分固定大小的内存当作分区来使用。它并非一个实际的文件系统，而是一种将实际的文件系统装入内存的机制，并且可以作为根文件系统。将一些经常被访问而又不会更改的文件(如只读的根文件系统)通过Ramdisk放在内存中，可以明显地提高系统的性能。

在Linux的启动阶段，initrd提供了一套机制，可以将内核映像和根文件系统一起载入内存。


#### ramfs/tmpfs
Ramfs是Linus Torvalds开发的一种基于内存的文件系统，工作于虚拟文件系统(VFS)层，不能格式化，可以创建多个，在创建时可以指定其最大能使用的内存大 小。(实际上，VFS本质上可看成一种内存文件系统，它统一了文件在内核中的表示方式，并对磁盘文件系统进行缓冲。)

Ramfs/tmpfs文件系统把所有的文件都放在RAM中，所以读/写操作发生在RAM中，可以用ramfs/tmpfs来存储一些临时性或经常要修改的数据，例如/tmp和/var目录，这样既避免了对Flash存储器的读写损耗，也提高了数据读写速度。

Ramfs/tmpfs相对于传统的Ramdisk的不同之处主要在于：不能格式化，文件系统大小可随所含文件内容大小变化。

Tmpfs的一个缺点是当系统重新引导时会丢失所有数据。



### 网络文件系统
NFS (Network File System)是由Sun开发并发展起来的一项在不同机器、不同操作系统之间通过网络共享文件的技术。在嵌入式Linux系统的开发调试阶段，可以利用该技术在主机上建立基于NFS的根文件系统，挂载到嵌入式设备，可以很方便地修改根文件系统的内容。

以上讨论的都是基于存储设备的文件系统(memory-based file system)，它们都可用作Linux的根文件系统。实际上，Linux还支持逻辑的或伪文件系统(logical or pseudo file system)，例如procfs(proc文件系统)，用于获取系统信息，以及devfs(设备文件系统)和sysfs，用于维护设备文件。


