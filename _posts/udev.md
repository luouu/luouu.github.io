<!--
 * @Description: udev
 * @Author: luo_u
 * @Date: 2020-07-08 21:28:19
 * @LastEditTime: 2020-09-10 10:56:05
--> 

[TOC]

## udev

udev 完全在用户态工作，利用设备加入或移除时内核所发送的热插拔事件(hotplug event)来工作。在热插拔时，设备的详细信息会由内核输出到位于/sys 的sysfs 文件系统。udev 的设备命名策略、权限控制和事件处理都是在用户态下完成的，它利用 sysfs 中的信息来进行创建设备文件节点等工作。

udev 根据系统中硬件设备的状态动态更新、创建和删除设备文件，进行设备文件的等，因此/dev 目录下只包含系统中真正存在的设备。linux在设备被发现的时候加载驱动模块。

udev主页 http://www.kernel.org/pub/linux/utils/kernel/hotplug/udev.html

udev包下载 http://www.us.kernel.org/pub/linux/utils/kernel/hotplug/

udev 的设计目标如下：

- 在用户空间中执行。
- 动态建立/删除设备文件。
- 允许每个人都不用关心主/次设备号。
- 提供 LSB 标准名称，如果需要，可提供固定的名称。

udev 以 3 个分割的子计划发展：namedev、libsysfs 和 udev。namedev 为设备命名子系统；libsysfs 提供访问 sysfs 文件系统从中获取信息的标准接口；udev 提供/dev 设备节点文件的动态创建和删除策略，负责与 namedev 和 libsysfs 库交互的任务，当/sbin/hotplug程序被内核调用时，udev 将被运行。udev 的工作过程如下：

1. 当内核检测到在系统中出现了新设备后，内核会在 sysfs 文件系统中为该新设备生成新的记录并导出一些设备特定的信息及所发生的事件。

2. udev 获取内核导出的信息，它调用 namedev 决定应该给该设备指定的名称，如果是新插入设备，udev 将调用 libsysfs 决定应该为该设备的设备文件指定的主/次设备号，并用分析获得的设备名称和主/次设备号创建/dev 中的设备文件;如果是设备移除，则之前已经被创建的/dev 文件将被删除。


namedev 中使用 5 步序列来决定指定设备的命名：

1. 标签(label)/序号(serial):这一步检查设备是否有惟一的识别记号，例如 USB 设备有惟一的 USB 序号，SCSI 有惟一的 UUID。如果 namedev 找到与这种惟一编号相对应的规则，它将使用该规则提供的名称。

2. 设备总线号:这一步会检查总线设备编号，对于不可热插拔的环境，这一步足以辨别设备。例如，PCI 总线编号在系统的使用期间内很少变更。如果 namedev 找到相对应的规则，规则中的名称就会被使用。

3. 总线上的拓扑:当设备在总线上的位置匹配用户指定的规则时，就会使用该规则指定的名称。

4. 替换名称:当内核提供的名称匹配指定的替代字符串时，就会使用替代字符串指定的名称。

5. 内核提供的名称:如果以前的几个步骤都没有被提供，默认的内核将被指定给该设备。



## udev规则文件
udev 的规则文件以行为单位，以“#”开头的行代表注释行。其余的每一行代表一个规则。每个规则分成一个或多个匹配和赋值部分。匹配部分用匹配专用的关键字来表示，相应的赋值部分用赋值专用的关键字来表示。匹配关键字包括:ACTION(行为)、KERNEL(匹配内核设备名)、BUS(匹配总线类型)、SYSFS(匹配从 sysfs 得到的信息，比如 label、vendor、USB 序列号)、SUBSYSTEM(匹配子系统名)等，赋值关键字包括:NAME(创建的设备文件名)、SYMLINK(符号创建链接名)、OWNER(设置设备的所有者)、GROUP(设置设备的组)、IMPORT(调用外部程序)等。

```shell
SUBSYSTEM=="net", ACTION=="add", SYSFS{address}=="00:0d:87:f6:59:f3", IMPORT="/sbin/rename_netiface %k eth0"
```



## udevadm

udevadm可用于控制服务、 请求内核事件、管理事件队列、进行简单的调试。

- `-d`, `--debug`

  在标准错误(STDERR)上显示调试信息。 **udevadm test** 与 **udevadm test-builtin** 命令隐含了此选项。

- `-h`, `--help`

  显示简短的帮助信息并退出。



### udevadm info [*options*] [*devpath*|*file*|*unit*...] 

从udev数据库中提取设备信息。

位置参数用于指定一个或多个设备，它可以是 一个设备名(必须以 `/dev/` 开头)、 一个 sys 路径(必须以 `/sys/` 开头)、 一个设备单元(必须以 "`.device`" 结尾)。详见 [systemd.device(5)](http://www.jinbuguo.com/systemd/systemd.device.html#) 手册。

- `-q`, `--query=*TYPE*`

  提取特定类型的设备信息。 *TYPE* 可以是下列值之一： `name`, `symlink`, `path`, `property`, `all`(默认值)

- `-p`, `--path=*DEVPATH*`

  该设备在 `/sys` 目录下的路径(例如 `[/sys]/class/block/sda`)。 因为此选项是位置参数以 `/sys/` 开头时的替代， 所以通常将 **udevadm info --path=/class/block/sda** 直接简写为 **udevadm info /sys/class/block/sda**

- `-n`, `--name=*FILE*`

  设备节点或软连接的名称(例如 `[/dev]/sda`)。 因为此选项是位置参数以 `/dev/` 开头时的替代， 所以通常将 **udevadm info --name=sda** 直接简写为 **udevadm info /dev/sda**

- `-r`, `--root`

  以绝对路径显示 `--query=name` 与 `--query=symlink` 的查询结果

- `-a`, `--attribute-walk`

  按照udev规则的格式，显示所有可用于匹配该设备的sysfs属性： 从该设备自身开始，沿着设备树向上回溯(一直到树根)， 显示沿途每个设备的sysfs属性。

- `-x`, `--export`

  以 键='值' 的格式输出此设备的属性(注意，值两边有单引号界定)。 仅在指定了 `--query=property` 或 `--device-id-of-file=*FILE*` 的情况下才有效。

- `-P`, `--export-prefix=*NAME*`

  在输出的键名前添加一个前缀。 此选项隐含了 `--export`

- `-d`, `--device-id-of-file=*FILE*`

  显示 *FILE* 文件所在底层设备的主/次设备号。 如果使用了此选项，那么将忽略所有位置参数。

- `-e`, `--export-db`

  导出udev数据库的全部内容

- `-c`, `--cleanup-db`

  清除udev数据库

- `-h`, `--help`

  显示简短的帮助信息并退出。



### udevadm trigger [*options*] [*devpath*|*file*|*unit*] 

强制内核触发设备事件，主要用于重放内核初始化过程中的冷插(coldplug)设备事件。

接受一个用于指定设备的位置参数。参见前面对 **info** 的描述。

- `-v`, `--verbose`

  显示被触发的设备列表

- `-n`, `--dry-run`

  并不真正触发设备事件

- `-t`, `--type=*TYPE*`

  仅触发特定类型的设备， TYPE 可以是下列值之一： **devices**(默认值), **subsystems**

- `-c`, `--action=*ACTION*`

  指定触发哪种类型的设备事件，ACTION 可以是下列值之一： **add**, **remove**, **change**(默认值)

- `-s`, `--subsystem-match=*SUBSYSTEM*`

  仅触发属于 *SUBSYSTEM* 子系统的设备事件。 可以在 *SUBSYSTEM* 中使用shell风格的通配符。 如果多次使用此选项，那么表示以 OR 逻辑连接每个匹配规则， 也就是说，所有匹配的子系统中的设备都会被触发。

- `-S`, `--subsystem-nomatch=*SUBSYSTEM*`

  不触发属于 *SUBSYSTEM* 子系统的设备事件。 可以在 *SUBSYSTEM* 中使用shell风格的通配符。 如果多次使用此选项，那么表示以 AND 逻辑连接每个匹配规则， 也就是说，只有不匹配所有指定子系统的设备才会被触发。

- `-a`, `--attr-match=*ATTRIBUTE*=*VALUE*`

  仅触发那些在设备的sysfs目录中存在 ATTRIBUTE 文件的设备事件。 如果同时还指定了"=VALUE"，那么表示仅触发那些 ATTRIBUTE 文件的内容匹配 VALUE 的设备事件。 注意，可以在 VALUE 中使用shell风格的通配符。 如果多次使用此选项，那么表示以 AND 逻辑连接每个匹配规则， 也就是说，只有匹配所有指定属性的设备才会被触发。

- `-A`, `--attr-nomatch=*ATTRIBUTE*=*VALUE*`

  不触发那些在设备的sysfs目录中存在 ATTRIBUTE 文件的设备事件。 如果同时还指定了"=VALUE"，那么表示不触发那些 ATTRIBUTE 文件的内容匹配 VALUE 的设备事件。 注意，可以在 VALUE 中使用shell风格的通配符。 如果多次使用此选项，那么表示以 AND 逻辑连接每个匹配规则， 也就是说，只有不匹配所有指定属性的设备才会被触发。

- `-p`, `--property-match=*PROPERTY*=*VALUE*`

  仅触发那些设备的 PROPERTY 属性值匹配 VALUE 的设备事件。注意，可以在 VALUE 中使用shell风格的通配符。 如果多次使用此选项，那么表示以 OR 逻辑连接每个匹配规则， 也就是说，匹配任意一个属性值的设备都会被触发。

- `-g`, `--tag-match=*PROPERTY*`

  仅触发匹配 PROPERTY 标签的设备事件。如果多次使用此选项， 那么表示以 AND 逻辑连接每个匹配规则，也就是说，只有匹配所有指定标签的设备才会被触发。

- `-y`, `--sysname-match=*SYSNAME*`

  仅触发设备sys名称(也就是该设备在 `/sys` 路径下最末端的文件名)匹配 *SYSNAME* 的设备事件。 注意，可以在 *SYSNAME* 中使用shell风格的通配符。 如果多次使用此选项，那么表示以 OR 逻辑连接每个匹配规则， 也就是说，匹配任意一个sys名称的设备都会被触发。

- `--name-match=*DEVPATH*`

  触发给定设备及其所有子设备的事件。*DEVPATH* 是该设备在 `/dev` 目录下的路径。 如果多次使用此选项，那么仅以最后一个为准。

- `-b`, `--parent-match=*SYSPATH*`

  触发给定设备及其所有子设备的事件。*SYSPATH* 是该设备在 `/sys` 目录下的路径。 如果多次使用此选项，那么仅以最后一个为准。

- `-w`, `--settle`

  除了触发设备事件之外，还要等待这些事件完成。 注意，此选项仅等待该命令自身触发的事件完成， 而 **udevadm settle** 则要一直等到 所有设备事件全部完成。

- `--wait-daemon[=*SECONDS*]`

  在触发设备事件之前，等待 systemd-udevd 守护进程完成初始化。 默认等待 5 秒之后超时(可以使用 *SECONDS* 参数修改)。 此选项等价于在 **udevadm trigger** 命令之前先使用 **udevadm control --ping** 命令。

- `-h`, `--help`

  显示简短的帮助信息并退出。

可以直接使用 以 `/sys` 或 `/dev` 开头的绝对路径来指定目标设备。



### udevadm settle [*options*] 

监视udev事件队列，并且在所有事件全部处理完成之后退出。

- `-t`, `--timeout=*SECONDS*`

  最多允许花多少秒等候事件队列清空。 默认值是120秒。 设为 0 表示仅检查事件队列是否为空， 并且立即返回。

- `-E`, `--exit-if-exists=*FILE*`

  如果 FILE 文件存在，则停止等待。

- `-h`, `--help`

  显示简短的帮助信息并退出。



### udevadm control *option*

控制udev守护进程(systemd-udevd)的内部状态。

- `-e`, `--exit`

  向 systemd-udevd 发送"退出"信号并等待其退出。因为 `systemd-udevd.service` 中含有 `Restart=always` ，所以此选项实际是重启了 systemd-udevd 。 如果你想停止 `systemd-udevd.service` ，那么应该使用：`systemctl stop systemd-udevd-control.socket systemd-udevd-kernel.socket systemd-udevd.service`

- `-l`, `--log-priority=*value*`

  设置 [systemd-udevd.service(8)](http://www.jinbuguo.com/systemd/systemd-udevd.service.html#) 的内部日志等级。 可以用数字或文本表示： r`emerg`(0), `alert`(1), `crit`(2), `err`(3), `warning`(4), `notice`(5), `info`(6), `debug`(7)

- `-s`, `--stop-exec-queue`

  向 systemd-udevd 发送"禁止处理事件"信号， 这样所有新发生的事件都将进入等候队列。

- `-S`, `--start-exec-queue`

  向 systemd-udevd 发送"开始处理事件"信号，也就是开始处理事件队列中尚未处理的事件。

- `-R`, `--reload`

  向 systemd-udevd 发送"重新加载"信号，也就是重新加载udev规则与各种数据库(包括内核模块索引)。 注意，重新加载之后并不影响已经存在的设备， 但是新的配置将会应用于所有将来发生的新设备事件。

- `-p`, `--property=*KEY*=*value*`

  为所有将来发生的新设备事件统一设置一个全局的 KEY 属性，并将其值设为 value

- `-m`, `--children-max=`*value*

  设置最多允许 systemd-udevd 同时处理多少个设备事件。

- `--ping`

  向 systemd-udevd 发送一个"ping"消息并等待应答。用于检查 systemd-udevd 守护进程是否仍在正常运行。

- `-t`, `--timeout=`*seconds*

  等候 systemd-udevd 应答的最大秒数。

- `-h`, `--help`

  显示简短的帮助信息并退出。



### udevadm monitor [*options*] 

监视内核发出的设备事件(以"KERNEL"标记)， 以及udev在处理完udev规则之后发出的事件(以"UDEV"标记)，并在控制台上输出事件的设备路径(devpath)。 可用于分析udev处理设备事件所花的时间(比较"KERNEL"与"UDEV"的时间戳)。

- `-k`, `--kernel`

  仅显示"KERNEL"事件

- `-u`, `--udev`

  仅显示"UDEV"事件

- `-p`, `--property`

  同时还显示事件的各属性

- `-s`, `--subsystem-match=*subsystem[/devtype]*`

  根据 subsystem[/devtype] 对事件(包括 kernel uevent 与 udev event)进行过滤，仅显示与"子系统[/设备类型]"匹配的"UDEV"事件。 如果多次使用此选项，那么表示以 OR 逻辑连接每个匹配规则， 也就是说，所有指定子系统中的设备都会被监视。

- `-t`, `--tag-match=*string*`

  根据设备标签对事件(仅 udev event)进行过滤，仅显示与"标签"匹配的"UDEV"事件。 如果多次使用此选项，那么表示以 OR 逻辑连接每个匹配规则， 也就是说，拥有任一指定标签的设备都会被监视。

- `-h`, `--help`

  显示简短的帮助信息并退出。



### udevadm test [*options*] [*devpath*] 

模拟一个设备事件，并输出调试信息。

- `-a`, `--action=*ACTION*`

  指定模拟哪种类型的设备事件，ACTION 可以是下列值之一：**add**(默认值), **remove**, **change**

- `-N`, `--resolve-names=early|late|never`

  指定 udevadm 何时解析用户与组的名称： `early`(默认值) 表示在规则的解析阶段； `late` 表示在每个事件发生的时候； `never` 表示从不解析， 所有设备的属主与属组都是 root 。

- `-h`, `--help`

  显示简短的帮助信息并退出。



### udevadm test-builtin [*options*] [*command*] [*devpath*] 

针对 *DEVPATH*设备 运行一个内置的 *COMMAND* 命令， 并输出调试信息。

- `-h`, `--help`

  显示简短的帮助信息并退出。



## mdev
mdev 是udev的轻量级版本，集成于 busybox中。为了使用mdev 功能，/etc/init.d/rcS 包含的如下内容:
```shell
/bin/mount -t sysfs sysfs /sys
/bin/mount -t tmpfs mdev /dev
echo /bin/mdev > /proc/sys/kernel/hotplug
mdev -s
```

`echo /sbin/mdev > /proc/sys/kernel/hotplug`的含义是当有热插拔事件产生时，内核会调用mdev。这时mdev扫描/sys 中所有的类设备目录，通过环境变量中的ACTION 和 DEVPATH，来确定此次热插拔事件的动作以及影响了/sys 中的那个目录。而`mdev -s`是自动扫描/sys 中所有的类设备目录。如果在目录中含有名为“dev”的文件，且文件中包含的是设备号，则mdev 就利用这些信息为该设备在/dev 下创建设备节点文件。

修改/etc/mdev.conf 文件可以修改mdev 的规则。



## 参考

http://www.jinbuguo.com/systemd/udevadm.html