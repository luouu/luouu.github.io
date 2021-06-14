开发一个自己的 Linux 发行版，从头开发一个 Linux 发行版这件事情被称作 Linux From Scratch （LFS）。LFS 手册是打造 LFS 的官方指南。官网：http://www.linuxfromscratch.org/index.html
手册在线访问地址：https://linux.cn/lfs/LFS-BOOK-7.7-systemd/index.html 。

##　目录介绍
### /etc
#### /etc/passwd
/etc/passwd该文件对所有用户可读。文件中的每个用户都有一个对应的记录行，记录着这个用户的一下基本属性，每行记录又被冒号(:)分隔为7个字段，其格式和具体含义如下：

**用户名:口令:用户标识号:组标识号:注释性描述:主目录:登录Shell**

- 用户名(login_name):是代表用户账号的字符串。通常长度不超过8个字符，并且由大小写字母和/或数字组成。登录名中不能有冒号(:)，因为冒号在这里是分隔符。为了兼容起见，登录名中最好不要包含点字符(.)，并且不使用连字符(-)和加号(+)打头。
- 口令(passwd):一些系统中，存放着加密后的用户口令字。虽然这个字段存放的只是用户口令的加密串，不是明文，但是由于/etc/passwd文件对所有用户都可读，所以这仍是一个安全隐患。因此，现在许多Linux系统（如SVR4）都使用了shadow技术，把真正的加密后的用户口令字存放到/etc/shadow文件中，而在/etc/passwd文件的口令字段中只存放一个特殊的字符，例如“x”或者“*”。
- 用户标识号(UID):是一个整数，系统内部用它来标识用户。一般情况下它与用户名是一一对应的。如果几个用户名对应的用户标识号是一样的，系统内部将把它们视为同一个用户，但是它们可以有不同的口令、不同的主目录以及不同的登录Shell等。取值范围是0-65535。0是超级用户root的标识号，1-99由系统保留，作为管理账号，普通用户的标识号从100开始。在Linux系统中，这个界限是500。
- 组标识号(GID):字段记录的是用户所属的用户组。它对应着/etc/group文件中的一条记录。
- 注释性描述(users):字段记录着用户的一些个人情况，例如用户的真实姓名、电话、地址等，这个字段并没有什么实际的用途。在不同的Linux系统中，这个字段的格式并没有统一。在许多Linux系统中，这个字段存放的是一段任意的注释性描述文字，用做finger命令的输出。
- 主目录(home_directory):也就是用户的起始工作目录，它是用户在登录到系统之后所处的目录。在大多数系统中，各用户的主目录都被组织在同一个特定的目录下，而用户主目录的名称就是该用户的登录名。各用户对自己的主目录有读、写、执行（搜索）权限，其他用户对此目录的访问权限则根据具体情况设置。
- 登录Shell(Shell):用户登录后，要启动一个进程，负责将用户的操作传给内核，这个进程是用户登录到系统后运行的命令解释器或某个特定的程序，即Shell。Shell是用户与Linux系统之间的接口。系统管理员可以根据系统情况和用户习惯为用户指定某个Shell。如果不指定Shell，那么系统使用sh为默认的登录Shell，即这个字段的值为/bin/sh。 

#### /etc/shadow

/etc/shadow文件中的记录行与/etc/passwd中的一一对应，是passwd文件的一个影子。它由pwconv命令根据/etc/passwd中的数据自动产生。但是/etc/shadow文件只有系统管理员才能够进行修改和查看。

**登录名:加密口令:最后一次修改时间:最小时间间隔:最大时间间隔:警告时间:不活动时间:失效时间:标志**

1. “登录名”是与/etc/passwd文件中的登录名相一致的用户账号
2. “口令”字段存放的是加密后的用户口令字，格式为`$id$salt$hashed`，则表示该用户密码正常。其中`$id$`的id表示密码的加密算法，`$salt$`是加密时使用的salt，hashed才是真正的密码部分。
```
如果为空，则对应用户没有口令，登录时不需要口令；
*代表帐号被锁定；
!!表示这个密码已经过期了；
$1$是用MD5加密；
$2a$ 是用Blowfish加密；
$2y$是另一算法长度的Blowfish
$5$ 是用 SHA-256加密；
$6$表示SHA-512算法
```

3. “最后一次修改时间”表示的是从某个时刻起，到用户最后一次修改口令时的天数。时间起点对不同的系统可能不一样。例如在SCOLinux中，这个时间起点是1970年1月1日。
4. “最小时间间隔”指的是两次修改口令之间所需的最小天数。
5. “最大时间间隔”指的是口令保持有效的最大天数。
6. “警告时间”字段表示的是从系统开始警告用户到用户密码正式失效之间的天数。
7. “不活动时间”表示的是用户没有登录活动但账号仍能保持有效的最大天数。
8. “失效时间”字段给出的是一个绝对的天数，如果使用了这个字段，那么就给出相应账号的生存期。期满后，该账号就不再是一个合法的账号，也就不能再用来登录了。

#### /etc/group

- 组名：组的名称，可以将其视为与数值型组标识符相对应的人类可读(符号)标识符。
- 组密码：这里的 "x" 仅仅是密码标识，真正加密后的组密码默认保存在 /etc/gshadow 文件中。
- 组 ID (GID)：该组的数值型 ID。正常情况下,对应于组 ID 号 0，只定义一个名为 root的组。
- 用户列表：属于该组的用户名列表,之间以逗号分隔。


### /dev
#### /dev/null
任何输入到这个“设备”的数据都将被直接丢弃。最常用的用法是把不需要的输出重定向到这个文件。

- 禁止标准输出，文件内容丢失，而不会输出到标准输出
>cat $filename >/dev/null

- 禁止标准错误
>cat $filename 2>/dev/null

- 禁止标准输出和标准错误的输出
>cat \$filename 2>/dev/null >/dev/null

>cat $filename &>/dev/null

- 清空文件的内容
>cat /dev/null > /var/log/messages

- 隐藏cookie而不再使用
>ln -s /dev/null ~/.netscape/cookies

#### /dev/zero
是一个伪文件，但它实际上产生连续不断的null的流（二进制的零流，而不是ASCII型的）。写入它的输出会丢失不见，/dev/zero主要的用处是用来创建一个指定长度用于初始化的空文件，像临时交换文件。


## 文件管理
###　权限
r(Read，读取)：对文件而言，具有读取文件内容的权限；对目录来说，具有浏览目录的权限。 

w(Write,写入)：对文件而言，具有新增、修改文件内容的权限；对目录来说，具有删除、移动目录内文件的权限。

x(execute，执行)：对文件而言，具有执行文件的权限；对目录了来说该用户具有进入目录的权限。

s或S（SUID,Set UID）：使文件在执行时临时拥有文件所有者的权限。在设置s权限时文件属主、属组必须先设置x权限，否则s权限不能生效。当我们ls -l时看到rwS，大写S说明s权限未生效。passwd便是个设置了SUID的程序，普通用户无读写/etc/shadow文件的权限也可以修改自己的密码。

t或T（Sticky）：t权限一般只用在目录上，任何的用户都能够在这个目录下创建文档，但只能删除自己创建的文档，这就对任何用户能写的目录下的用户文档启到了保护的作用。可通过`chmod +t` 　`chmod -t` 　`chmod 1777`来设置。

i不可修改权限。通过`chattr +i` 　`chattr -i`来设置，用`lsattr`命令查看。

a只追加权限， 对于日志系统很好用，让目标文件只能追加，不能删除，而且不能通过编辑器追加。通过`chattr +a` 　`chattr -a`来设置。

## 启动流程

###　init进程

首先，在内核init/main.c文件中，init_post函数完成了对根文件系统和控制台的检测，并启动init进程。

```c
static noinline int init_post(void)
{
	/* need to finish all async __init code before freeing the memory */
	async_synchronize_full();
	free_initmem();
	mark_rodata_ro();
	system_state = SYSTEM_RUNNING;
	numa_default_policy();


	current->signal->flags |= SIGNAL_UNKILLABLE;

	if (ramdisk_execute_command) {
		run_init_process(ramdisk_execute_command);
		printk(KERN_WARNING "Failed to execute %s\n",
				ramdisk_execute_command);
	}

	/*
	 * We try each of these until one succeeds.
	 *
	 * The Bourne shell can be used instead of init if we are
	 * trying to recover a really broken machine.
	 */
	if (execute_command) {
		run_init_process(execute_command);
		printk(KERN_WARNING "Failed to execute %s.  Attempting "
					"defaults...\n", execute_command);
	}
	run_init_process("/sbin/init");
	run_init_process("/etc/init");
	run_init_process("/bin/init");
	run_init_process("/bin/sh");

	panic("No init found.  Try passing init= option to kernel. "
	      "See Linux Documentation/init.txt for guidance.");
}
```
内核会先根据uboot参数`init=/sbin/init`来启动第一个进程，一般都是init，实质是指向busybox的软链接。

init进程的入口是busybox/init/init.c中的init_main()函数。
```c
	/* Check if we are supposed to be in single user mode */
	if (argv[1]
	 && (strcmp(argv[1], "single") == 0 || strcmp(argv[1], "-s") == 0 || LONE_CHAR(argv[1], '1'))
	) {
		/* ??? shouldn't we set RUNLEVEL="b" here? */
		/* Start a shell on console */
		new_init_action(RESPAWN, bb_default_login_shell, "");
	} else {
		/* Not in single user mode - see what inittab says */

		/* NOTE that if CONFIG_FEATURE_USE_INITTAB is NOT defined,
		 * then parse_inittab() simply adds in some default
		 * actions (i.e., INIT_SCRIPT and a pair
		 * of "askfirst" shells) */
		parse_inittab();
	}
```

因为内核启动init进程没有传入附加参数，所以argv[1]不存在，程序走parse_inittab()。如果没有定义`CONFIG_FEATURE_USE_INITTAB` 这个宏，程序会执行一些默认的action，这个宏是busybox配置时的选项。最后调用run_action()运行每一个action，并且首先运行的是action为sysinit的动作。

```ini
config FEATURE_USE_INITTAB
	bool "Support reading an inittab file"
	default y
	depends on INIT || LINUXRC
	help
	Allow init to read an inittab file when the system boot.
```
busybox/init/init.c中的parse_inittab()函数的定义如下：
```c
/* NOTE that if CONFIG_FEATURE_USE_INITTAB is NOT defined,
 * then parse_inittab() simply adds in some default
 * actions (i.e., runs INIT_SCRIPT and then starts a pair
 * of "askfirst" shells).  If CONFIG_FEATURE_USE_INITTAB
 * _is_ defined, but /etc/inittab is missing, this
 * results in the same set of default behaviors.
 */
static void parse_inittab(void)
{
#if ENABLE_FEATURE_USE_INITTAB
	char *token[4];
	parser_t *parser = config_open2("/etc/inittab", fopen_for_read);

	if (parser == NULL)
#endif
	{
		/* No inittab file - set up some default behavior */
		/* Sysinit */
		new_init_action(SYSINIT, INIT_SCRIPT, "");
		/* Askfirst shell on tty1-4 */
		new_init_action(ASKFIRST, bb_default_login_shell, "");
//TODO: VC_1 instead of ""? "" is console -> ctty problems -> angry users
		new_init_action(ASKFIRST, bb_default_login_shell, VC_2);
		new_init_action(ASKFIRST, bb_default_login_shell, VC_3);
		new_init_action(ASKFIRST, bb_default_login_shell, VC_4);
		/* Reboot on Ctrl-Alt-Del */
		new_init_action(CTRLALTDEL, "reboot", "");
		/* Umount all filesystems on halt/reboot */
		new_init_action(SHUTDOWN, "umount -a -r", "");
		/* Swapoff on halt/reboot */
		new_init_action(SHUTDOWN, "swapoff -a", "");
		/* Restart init when a QUIT is received */
		new_init_action(RESTART, "init", "");
		return;
	}

#if ENABLE_FEATURE_USE_INITTAB
	/* optional_tty:ignored_runlevel:action:command
	 * Delims are not to be collapsed and need exactly 4 tokens
	 */
	while (config_read(parser, token, 4, 0, "#:",
				PARSE_NORMAL & ~(PARSE_TRIM | PARSE_COLLAPSE))) {
		/* order must correspond to SYSINIT..RESTART constants */
		static const char actions[] ALIGN1 =
			"sysinit\0""wait\0""once\0""respawn\0""askfirst\0"
			"ctrlaltdel\0""shutdown\0""restart\0";
		int action;
		char *tty = token[0];

		if (!token[3]) /* less than 4 tokens */
			goto bad_entry;
		action = index_in_strings(actions, token[2]);
		if (action < 0 || !token[3][0]) /* token[3]: command */
			goto bad_entry;
		/* turn .*TTY -> /dev/TTY */
		if (tty[0]) {
			tty = concat_path_file("/dev/", skip_dev_pfx(tty));
		}
		new_init_action(1 << action, token[3], tty);
		if (tty[0])
			free(tty);
		continue;
 bad_entry:
		message(L_LOG | L_CONSOLE, "Bad inittab entry at line %d",
				parser->lineno);
	}
	config_close(parser);
#endif
}
```
首先分析/etc/inittab文件，如果不存在inittab文件，则执行默认的action；如果inittab文件存在，则解析/etc/inittab文件的内容，actions[]数组的内容对应着inittab文件中的条目。

###　inittab文件

busybox/examples/下面有inittab脚本的例程

```shell
# /etc/inittab init(8) configuration for BusyBox
#
# Copyright (C) 1999-2004 by Erik Andersen <andersen@codepoet.org>
#
# Note, BusyBox init doesn't support runlevels.  The runlevels field is
# completely ignored by BusyBox init. If you want runlevels, use sysvinit.
#
#
# Format for each entry: <id>:<runlevels>:<action>:<process>
#
# <id>: WARNING: This field has a non-traditional meaning for BusyBox init!
#
#	The id field is used by BusyBox init to specify the controlling tty for
#	the specified process to run on.  The contents of this field are
#	appended to "/dev/" and used as-is.  There is no need for this field to
#	be unique, although if it isn't you may have strange results.  If this
#	field is left blank, then the init's stdin/out will be used.
#
# <runlevels>: The runlevels field is completely ignored.
#
# <action>: Valid actions include: sysinit, respawn, askfirst, wait, once,
#                                  restart, ctrlaltdel, and shutdown.
#
#       Note: askfirst acts just like respawn, but before running the specified
#       process it displays the line "Please press Enter to activate this
#       console." and then waits for the user to press enter before starting
#       the specified process.
#
#       Note: unrecognized actions (like initdefault) will cause init to emit
#       an error message, and then go along with its business.
#
# <process>: Specifies the process to be executed and it's command line.
#
# Note: BusyBox init works just fine without an inittab. If no inittab is
# found, it has the following default behavior:
#         ::sysinit:/etc/init.d/rcS
#         ::askfirst:/bin/sh
#         ::ctrlaltdel:/sbin/reboot
#         ::shutdown:/sbin/swapoff -a
#         ::shutdown:/bin/umount -a -r
#         ::restart:/sbin/init
#         tty2::askfirst:/bin/sh
#         tty3::askfirst:/bin/sh
#         tty4::askfirst:/bin/sh
#
# Boot-time system configuration/initialization script.
# This is run first except when booting in single-user mode.
#
::sysinit:/etc/init.d/rcS

# /bin/sh invocations on selected ttys
#
# Note below that we prefix the shell commands with a "-" to indicate to the
# shell that it is supposed to be a login shell.  Normally this is handled by
# login, but since we are bypassing login in this case, BusyBox lets you do
# this yourself...
#
# Start an "askfirst" shell on the console (whatever that may be)
::askfirst:-/bin/sh
# Start an "askfirst" shell on /dev/tty2-4
tty2::askfirst:-/bin/sh
tty3::askfirst:-/bin/sh
tty4::askfirst:-/bin/sh

# /sbin/getty invocations for selected ttys
tty4::respawn:/sbin/getty 38400 tty5
tty5::respawn:/sbin/getty 38400 tty6

# Example of how to put a getty on a serial line (for a terminal)
#::respawn:/sbin/getty -L ttyS0 9600 vt100
#::respawn:/sbin/getty -L ttyS1 9600 vt100
#
# Example how to put a getty on a modem line.
#::respawn:/sbin/getty 57600 ttyS2

# Stuff to do when restarting the init process
::restart:/sbin/init

# Stuff to do before rebooting
::ctrlaltdel:/sbin/reboot
::shutdown:/bin/umount -a -r
::shutdown:/sbin/swapoff -a
```

首先会去执行/etc/init.d/rcS脚本，然后启动shell到console口，开启getty，指定init进程的重启位置，处理重启之前要做的事情。

###　rcS脚本


## 配置
###　/etc配置
####  省略密码验证
把/etc/passwd中的`root:x:0:0:root:/root:/bin/bash`，改为`root::0:0:root:/root:/bin/bash`，就可以了，就是去掉了里面的x，这样root用户就不用密码了。其他用户也一样。

#### 省略输入用户名
修改/etc/inittab
```shell
1:2345:respawn:/sbin/agetty tty1 9600
2:2345:respawn:/sbin/agetty tty2 9600
3:2345:respawn:/sbin/agetty tty3 9600
4:2345:respawn:/sbin/agetty tty4 9600
5:2345:respawn:/sbin/agetty tty5 9600
6:2345:respawn:/sbin/agetty tty6 9600
```
表示系统可以有六个控制台，可以用ALT+(F1~F6)来切换。而/sbin/agetty就是一个登陆验证程序，执行它，会提示用户输入用户名和密码，然后启动一个指定的shell（在passwd文件中指定的）。所以，我们只需将其修改为不执行agettty，而是执行自己编写的一个脚本，就可以跳过用户名和密码的输入。修改如下：
>1:2345:respawn:/bin/autologin tty1 9600

autologin脚本内容：
```shell
#!/bin/sh
/bin/login -f root
```
####　修改主机名
- 临时修改主机名
> hostname 新主机名
- 永久修改主机名
1. 修改`/etc/hostname`文件，有的Linux发行版将主机名存放在`/etc/sysconfig/network`文件中。修改`/etc/hosts`配置文件

### core dump
- 设置core文件大小
1.　`ulimit -c 1024`，默认为0，即不生成，只针对当前shell
2.　永久设置，修改/etc/security/limits.conf文件：
```
#*               soft    core            0
```
修改成：
```
*                soft    core            unlimited
```

- 查询core dump文件路径

```shell
cat /proc/sys/kernel/core_pattern
/sbin/sysctl kernel.core_pattern
```

- 修改core dump文件路径
1.　临时修改
```shell
echo "/tmp/core_%e_%p_%t" > /proc/sys/kernel/core_pattern
```
2.　永久修改
```shell
/sbin/sysctl -w kernel.core_pattern=/var/log/%e.core.%p
```
3. 永久设置, 修改`/etc/sysctl.conf`配置文件，添加一行，然后执行`sysctl -p`
```
kernel.core_pattern = %e.core.%p
```
可通过以下参数来丰富core文件的命名：
```
%% 单个%字符
%p 所dump进程的进程ID
%u 所dump进程的实际用户ID
%g 所dump进程的实际组ID
%s 导致本次core dump的信号
%t core dump的时间 (由1970年1月1日计起的秒数)
%h 主机名
%e 程序文件名
```

-　core文件带pid
1. 修改`/proc/sys/kernel/core_uses_pid`，如果这个文件的内容被配置成1，即使core_pattern中没有设置%p，最后生成的core dump文件名仍会加上进程ID。
2.　永久设置, 修改/etc/sysctl.conf配置文件，添加一行，然后执行`sysctl -p`
```
kernel.core_uses_pid = 1
```
*注意*：
1.　自定义core dump文件路径时需要注意配置好路径的权限。
2.　在普通用户运行设置了setuid的程序，一定要将`/proc/sys/fs/suid_dumpable`设置为2才能生成coredump文件。



### /proc/sys/vm/
- block_dump
该文件表示是否打开 Block Debug 模式，用于记录所有的读写及 Dirty Block 写回动作缺省设置：0，禁用 Block Debug 模式

- dirty_background_ratio
该文件表示脏数据到达系统整体内存的百分比，此时触发 pdflush 进程把脏数据写回磁盘。缺省设置：10 

- dirty_expire_centisecs
该文件表示如果脏数据在内存中驻留时间超过该值，pdflush 进程在下一次将把这些数据写回磁盘。缺省设置：3000（1/100 秒） 

- dirty_ratio
该文件表示如果进程产生的脏数据到达系统整体内存的百分比，此时进程自行把脏数据写回磁盘。缺省设置：40 

- dirty_writeback_centisecs
该文件表示 pdflush 进程周期性间隔多久把脏数据写回磁盘。缺省设置：500（1/100 秒）

- vfs_cache_pressure
该文件表示内核回收用于 directory 和 inode cache 内存的倾向； 缺省值 100 表示内核将根据 pagecache 和 swapcache，把 directory 和 inode cache 保持在一个合理的百分比；降低该值低于 100，将导致内 核倾向于保留 directory 和 inode cache； 增加该值超过 100， 将导致内核倾向于回收 directory 和 inode cache缺省设置：100

- min_free_kbytes
该文件表示强制 Linux VM 最低保留多少空闲内存（Kbytes）。缺省设置：724（512M 物理内存） 

- nr_pdflush_threads
该文件表示当前正在运行的 pdflush 进程数量，在 I/O 负载高的情况下，内核会自动增加更多的 pdflush 进程。缺省设置：2（只读）

- overcommit_memory
该文件指定了内核针对内存分配的策略，其值可以是 0、1、2。0，表示内核将检查是否有足够的可用内存供应用进程使用； 如果有足够的可用内存， 内存申请允许； 否则， 内存申请失败，并把错误返回给应用进程。 1，表示内核允许分配所有的物理内存，而不管当前的内存状态如何。 2，表示内核允许分配超过所有物理内存和交换空间总和的内存（参照 overcommit_ratio）。缺省设置：0 

- overcommit_ratio 
该文件表示，如果 overcommit_memory=2，可以过载内存的百分比，通过以下公式来计算系统整体可用内存。 系统可分配内存=交换空间+物理内存*overcommit_ratio/100缺省设置：50（%） 

- page-cluster 
该文件表示在写一次到 swap 区的时候写入的页面数量，0 表示 1 页，1 表示 2 页，2 表示 4 页。缺省设置：3（2 的 3 次方，8 页）

 - swapiness 
 该文件表示系统进行交换行为的程度，数值（0-100）越高，越可能发生磁盘交换。缺省设置：60 

- legacy_va_layout 
该文件表示是否使用最新的 32 位共享内存 mmap()系统调用，Linux 支持的共享内存分配方式包括 mmap()，Posix，System VIPC。0，使用最新 32 位 mmap()系统调用。1，使用 2.4 内核提供的系统调用。缺省设置：0 

- nr_hugepages 
该文件表示系统保留的 hugetlb 页数。 

- hugetlb_shm_group 
该文件表示允许使用 hugetlb 页创建 System VIPC 共享内存段的系统组 ID。 


### 网络
####　查看和设置MTU
>ifconfig eth0
>
>ifconfig eth0 mtu 1500

>cat /sys/class/net/eth0/mtu
>
>echo "1400" > /sys/class/net/eth0/mtu



## 服务器搭建
###  SSH服务
>sudo apt-get install openssh-server

安装完成后，输入以下命令，用于检测是否安装并成功启动SSH：
>sudo ps -e|grep ssh

如果有 sshd 字样，说明服务已经启动，如果没有则可以通过以下命令启动SSH服务：
>sudo service ssh start

### Samba
1.　安装Samba
>sudo apt-get install samba samba-common

2.　创建分享目录，配置samba，修改`/etc/samba/smb.conf`，用`testparm`命令检查配置文件的语法错误。
```
[global]
	allow insecure wide links = yes


[homes]
	create mask = 0775
		directory mask = 0775
		valid users = %S
		read only = no
		writable = yes
		browseable = no
		follow symlinks = yes
		wide links = yes


[share]
	comment = share folder
		path = /share
		create mask = 0777
		directory mask = 0777
		read only = no
		writable = yes
		public = yes
		browseable = yes
```

3. 添加登入共享文件夹的用户名和密码，其中用户名必须为linux中的用户。
>sudo smbpasswd -a root

4.　重启samba服务
>service smbd restart

### NFS
1.　安装nfs相关软件
>sudo apt-get install nfs-kernel-server nfs-common

2. 配置nfs服务
>sudo vim /etc/exports
```
/nfsroot *(rw,sync,no_root_squash,no_subtree_check)
```

-　/nfsroot  ：共享的目录

- \*：指定哪些用户可以访问，
\*所有可以ping同该主机的用户；
192.168.1.\* 指定网段，在该网段中的用户可以挂载；
192.168.1.12 只有该用户能挂载；

-　(ro,sync,no_root_squash)： 权限，
ro : 只读；
rw : 读写；
sync : 数据同步写入内存和硬盘；
no_root_squash: 不降低root用户的权限；    

3. 重新启动nfs服务
>sudo service nfs-kernel-server restart




## 快捷键

`ctrl+c` 发送 SIGINT 信号给前台进程组中的所有进程。常用于终止正在运行的程序。

`ctrl+z` 发送 SIGTSTP 信号给前台进程组中的所有进程，常用于挂起一个进程，进程后台运行。

`ctrl+d` 不是发送信号，而是表示一个特殊的二进制值，表示 EOF。结束当前输入

`ctrl+\` 发送 SIGQUIT 信号给前台进程组中的所有进程，终止前台进程并生成 core 文件。

