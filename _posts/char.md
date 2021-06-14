<!--
 * @Description: 字符设备
 * @Author: luo_u
 * @Date: 2020-06-01 10:43:44
 * @LastEditTime: 2020-09-11 21:40:46
--> 

[TOC]

字符设备是一个顺序的数据流设备，对设备的读写是按字符进行的，而且这些字符是连续地形成一个数据流。不具备缓冲区，所以对字符设备的读写是实时的。



## 设备号

设备号分为主设备号和次设备号，前者为dev_t 的高12位，后者为dev_t 的低20位。inode 结构i_rdev 字段包含设备号。内核Documents 目录下的devices.txt 文件描述了linux设备号的分配情况。

主设备号用来标识设备类型， 次设备号用来区分同类型的设备。

```c
/*分解出主设备号*/
#define MAJOR(dev)	((dev)>>8)

/*分解出次设备号*/
#define MINOR(dev)	((dev) & 0xff)

/*合成dev_t*/
#define MKDEV(ma,mi)	((ma)<<8 | (mi))


/* 静态注册，设备号已知 */
int register_chrdev_region(dev_t from, unsigned count, const char *name);

/* 动态注册，设备号未知 */
int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name);

/*注销设备号*/
void unregister_chrdev_region(dev_t from, unsigned count);
```

可以通过读取/proc/devices文件获取字符设备和块设备的设备号信息。
```
Character devices:
  4 tty
  5 /dev/tty
  5 /dev/console
 10 misc
 13 input
 89 i2c
180 usb
249 watchdog
250 rtc
254 gpiochip

Block devices:
259 blkext
  7 loop
  8 sd
  9 md
 11 sr
 65 sd
```



## cdev
![cdev-file_ops](.img/char/cdev-file_ops.png)

```c
struct cdev {
	struct kobject kobj; /* 内嵌的 kobject 对象 */
	struct module *owner; /*所属模块*/
	struct file_operations *ops; /*文件操作结构体*/
	struct list_head list;
	dev_t dev; /*设备号*/
	unsigned int count;
};

struct cdev *cdev_alloc(void)
{
	struct cdev *p = kzalloc(sizeof(struct cdev), GFP_KERNEL);
	if (p) {
		INIT_LIST_HEAD(&p->list);
		kobject_init(&p->kobj, &ktype_cdev_dynamic);
	}
	return p;
}

void cdev_init(struct cdev *cdev, const struct file_operations *fops)
{
	memset(cdev, 0, sizeof *cdev);
	INIT_LIST_HEAD(&cdev->list);
	kobject_init(&cdev->kobj, &ktype_cdev_default);
	cdev->ops = fops;
}

int cdev_add(struct cdev *p, dev_t dev, unsigned count)
{
	int error;

	p->dev = dev;
	p->count = count;

	error = kobj_map(cdev_map, dev, count, NULL,
			 exact_match, exact_lock, p);
	if (error)
		return error;

	kobject_get(p->kobj.parent);

	return 0;
}


/* 注销字符设备 */
void cdev_del(struct cdev *p)
{
	cdev_unmap(p->dev, p->count);
	kobject_put(&p->kobj);
}

void cdev_put(struct cdev *p)
{
	if (p) {
		struct module *owner = p->owner;
		kobject_put(&p->kobj);
		module_put(owner);
	}
}

```
注册字符设备时先用 cdev_alloc()函数分配一个cdev结构体，然后使用 cdev_init()函数初始化cdev和对应的操作函数fops，最后用 cdev_add()函数添加字符设备到内核，要使用之前创建好的设备号。

```c
if(major)
	/*静态注册设备号*/
	register_chrdev_region(chardev->devno, 1, CHARDEV_DEV_NAME);
else
	/*动态注册设备号*/
	alloc_chrdev_region(&chardev->devno, 0, 1, CHARDEV_DEV_NAME);

cdev_init(&chardev->cdev, &chardev_fops);
chardev->cdev.owner = THIS_MODULE;
cdev_add(&chardev->cdev, chardev->devno, 1);

/* 在/sys/class/目录下创建设备 */
chardev->class = class_create(THIS_MODULE, CHARDEV_DEV_NAME);

/* 创建设备节点 */	
device_create(chardev->class, NULL, chardev->devno, NULL, CHARDEV_DEV_NAME);
```



## 设备节点

class_create()在/sys/class/目录下创建设备类。再调用 device_create()函数来在/dev目录下创建相应的设备节点。当加载模块时，用户空间中的udev会自动去/sysfs下寻找对应的类从而创建设备节点。

`udevadm info`命令可以查看/dev下的设备节点信息。

可以通过`mknod`命令来手动创建设备节点。
```shell
mknod [选项] name {bcp} major minor

选项： 
-Z：设置安全的上下文;
-m：设置权限模式;
-help：显示帮助信息;
--version：显示版本信息;

参数:
b 创建块设备;
c 创建字符设备;
p 创建命名管道;
```

## file
file 结构体代表一个打开的设备文件，系统中每个打开的文件在内核空间都有一个关联的 struct file，它由内核在打开文件时创建，在文件关闭后释放。

```c
struct file {
	union {
		struct list_head	fu_list;
		struct rcu_head 	fu_rcuhead;
	} f_u;

/*文件对应的目录项结构。除了用filp->f_dentry->d_inode的方式来访问索引节点结构*/
	struct path		f_path;
#define f_dentry	f_path.dentry
#define f_vfsmnt	f_path.mnt
	const struct file_operations	*f_op; /*文件相关的操作*/
	spinlock_t		f_lock;  /* f_ep_links, f_flags, no IRQ */
	atomic_long_t		f_count;
	unsigned int 		f_flags;  /*文件标志。如O_RONLY、O_NONBLOCK和O_SYNC*/
	fmode_t			f_mode;  /*文件模式。FMODE_READ和FMODE_WRITE分别表示读写权限*/
	loff_t			f_pos; /*当前的读写位置*/
	struct fown_struct	f_owner;
	const struct cred	*f_cred;
	struct file_ra_state	f_ra;
	u64			f_version;

	/* needed for tty driver, and maybe others */
	void			*private_data;  /*文件私有数据*/
	struct address_space	*f_mapping;
};

```



## file_operations

应用程序和VFS之间的接口是系统调用，而VFS与磁盘文件系统以及普通设备之间的接口是file_operations 结构体成员函数。file_operations 是对设备操作的抽象结构体，应用程序进行open()、write()、read()、close()等系统调用，最终都会引起file_operations 结构体对应函数的调用。
```c
struct file_operations {
	struct module *owner;
	loff_t (*llseek) (struct file *, loff_t, int);
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
	ssize_t (*aio_read) (struct kiocb *, const struct iovec *, unsigned long, loff_t);
	ssize_t (*aio_write) (struct kiocb *, const struct iovec *, 
                          unsigned long, loff_t);
	int (*readdir) (struct file *, void *, filldir_t);
	unsigned int (*poll) (struct file *, struct poll_table_struct *);
	long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
	long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
	int (*mmap) (struct file *, struct vm_area_struct *);
	int (*open) (struct inode *, struct file *);
	int (*flush) (struct file *, fl_owner_t id);
	int (*release) (struct inode *, struct file *);
	int (*fsync) (struct file *, int datasync);
	int (*aio_fsync) (struct kiocb *, int datasync);
	int (*fasync) (int, struct file *, int);
	int (*lock) (struct file *, int, struct file_lock *);
	ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
	unsigned long (*get_unmapped_area)(struct file *, unsigned long, 
                     unsigned long, unsigned long, unsigned long);
	
	int (*check_flags)(int);
	int (*flock) (struct file *, int, struct file_lock *);
	ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, 
                            loff_t *, size_t, unsigned int);
	ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, 
                           size_t, unsigned int);
	int (*setlease)(struct file *, long, struct file_lock **);
	long (*fallocate)(struct file *file, int mode, loff_t offset, loff_t len);

#if defined(CONFIG_SMM6260_MODEM) || defined(CONFIG_USE_GPIO_AS_I2C)
	int (*ioctl) (struct inode *, struct file *, unsigned int, unsigned long);
#endif
};
```
- mmap()函数将设备内存映射到进程内存中。

- poll()函数用于询问设备是否可被非阻塞地立即读写。当询问的条件未触发时，用户空间进行 select()和 poll()系统调用将引起进程的阻塞。

- aio_read()和 aio_write()函数分别对与文件描述符对应的设备进行异步读、写操作。

  


## inode
用来记录文件的物理上的信息，包含文件访问权限、属主、组、大小、生成时间、访问时间、最后修改时间等信息。一个文件可以对应多个file结构，但只有一个inode 结构。它是Linux管理文件系统的最基本单位，也是文件系统连接任何子目录、文件的桥梁。

```c
struct inode {
	umode_t			i_mode;  /* inode 的权限 */
	uid_t			i_uid;  /* inode 拥有者的 id */
	gid_t			i_gid;  /* inode 所属的群组 id */
	const struct inode_operations	*i_op;
	struct super_block	*i_sb;

	spinlock_t		i_lock;	/* i_blocks, i_bytes, maybe i_size */
	unsigned int		i_flags;
	unsigned long		i_state;
	struct mutex		i_mutex;

	unsigned long		dirtied_when;	/* jiffies of first dirtying */

	struct hlist_node	i_hash;
	struct list_head	i_wb_list;	/* backing dev IO list */
	struct list_head	i_lru;		/* inode LRU list */
	struct list_head	i_sb_list;
	union {
		struct list_head	i_dentry;
		struct rcu_head		i_rcu;
	};
	unsigned long		i_ino;
	atomic_t		i_count;
	unsigned int		i_nlink;
	dev_t			i_rdev;  /* 若是设备文件,此字段将记录设备的设备号 */
	unsigned int		i_blkbits;
	u64			i_version;
	loff_t			i_size;  /* inode 所代表的文件大小 */

	struct timespec i_atime; /* inode 最近一次的存取时间 */
	struct timespec i_mtime; /* inode 最近一次的修改时间 */
	struct timespec i_ctime; /* inode 的产生时间 */

	blkcnt_t		i_blocks;  /* inode 所使用的 block 数,一个 block为512 byte */
	unsigned short          i_bytes;

	struct rw_semaphore	i_alloc_sem;
	const struct file_operations	*i_fop;	/* former ->i_op->default_file_ops */
	struct file_lock	*i_flock;
	struct address_space	*i_mapping;
	struct address_space	i_data;

	struct list_head	i_devices;
	union {
		struct pipe_inode_info	*i_pipe;
		struct block_device	*i_bdev;  /*若是块设备,为其对应的 block_device 结构体指针*/
		struct cdev		*i_cdev;  /*若是字符设备,为其对应的 cdev 结构体指针*/
	};

	__u32			i_generation;

	void			*i_private; /* fs or device private pointer */
};
```
应用程序每打开一个设备文件，就会产生一个inode节点，通过结点的i_cdev字段找到cdev结构体，再通过cdev的ops指针，就能找到设备的操作函数。

i_rdev 表示设备文件对应的设备号，i_devices 和cdev的 list 相关联。

```c
int light_open(struct inode *inode, struct file *filp)
{
	struct light_dev *dev;
	/* 获得设备结构体指针 */
	dev = container_of(inode->i_cdev, struct light_dev, cdev);
	/* 让设备结构体作为设备的私有信息 */
	filp->private_data = dev;
	return 0;
}
```



## ioctl

Ioctl函数的实现通常是根据命令执行的一个switch语句，当命令都不匹配时，通常返回-EINVAL。

### _IORW
ioctl() 函数的cmd参数是应用程序用于区别设备驱动程序请求处理内容的值。 cmd的大小为 32位，共分 4 个域。

- **bit[31:30]**: 区别读写命令，值可能是 `_IOC_NONE` 表示无数据传输，`_IOC_READ` (读)， `_IOC_WRITE` (写) ， `_IOC_READ|_IOC_WRITE` (双向)。

- **bit[29:15]**: 数据大小，表示 ioctl() 中的 arg 变量传送的内存大小。

- **bit[20:8]**: 魔数，这个值用以与其它设备驱动程序的 ioctl 命令进行区别，魔数范围为 0~255 。通常用英文字符 "A" ~ "Z" 或者 "a" ~ "z" 来表示。

- **bit[7:0]**: 区别序号，是命令的顺序序号。

Doucumention/ioctl-number.txt文件中罗列了内核所使用的幻数，选择自己的幻数要避免和内核冲突。内核定义了 _IO() , _IOR() , IOW() 和 _IOWR() 这 4 个宏来辅助生成上面的 cmd 。

```c
// 序数（number）字段的字位宽度，8bits
#define _IOC_NRBITS      8 
#define _IOC_TYPEBITS    8 
#define _IOC_SIZEBITS    14 
#define _IOC_DIRBITS     2 

#define         _IOC_NRMASK        ((1 << _IOC_NRBITS)-1)  
#define         _IOC_TYPEMASK   ((1 << _IOC_TYPEBITS)-1) 
#define         _IOC_SIZEMASK     ((1 << _IOC_SIZEBITS)-1)  
#define         _IOC_DIRMASK      ((1 << _IOC_DIRBITS)-1)

#define        _IOC_NRSHIFT       0  
#define        _IOC_TYPESHIFT   (_IOC_NRSHIFT+_IOC_NRBITS) 
#define        _IOC_SIZESHIFT    (_IOC_TYPESHIFT+_IOC_TYPEBITS) 
#define        _IOC_DIRSHIFT      (_IOC_SIZESHIFT+_IOC_SIZEBITS)  

#define _IOC_NONE     0U  
#define _IOC_WRITE   1U   
#define _IOC_READ     2U 

/*_IOC 宏将dir，type，nr，size四个参数组合成一个cmd参数*/
#define _IOC(dir,type,nr,size) \

       (((dir)  << _IOC_DIRSHIFT) | \

        ((type) << _IOC_TYPESHIFT) | \

        ((nr)   << _IOC_NRSHIFT) | \

        ((size) << _IOC_SIZESHIFT))

//构造无参数的命令编号
#define _IO(type,nr)          _IOC(_IOC_NONE,(type),(nr),0)

//构造从驱动程序中读取数据的命令编号
#define _IOR(type,nr,size)    _IOC(_IOC_READ,(type),(nr),sizeof(size)) 

//用于向驱动程序写入数据命令
#define _IOW(type,nr,size)    _IOC(_IOC_WRITE,(type),(nr),sizeof(size))

//用于双向传输
#define _IOWR(type,nr,size)   _IOC(_IOC_READ|_IOC_WRITE,(type),(nr),sizeof(size))

//从命令参数中解析出数据方向，即写进还是读出
#define _IOC_DIR(nr)    (((nr) >> _IOC_DIRSHIFT) & _IOC_DIRMASK)

//从命令参数中解析出幻数type
#define _IOC_TYPE(nr)   (((nr) >> _IOC_TYPESHIFT) & _IOC_TYPEMASK)

//从命令参数中解析出序数number
#define _IOC_NR(nr)           (((nr) >> _IOC_NRSHIFT) & _IOC_NRMASK)

//从命令参数中解析出用户数据大小
#define _IOC_SIZE(nr)     (((nr) >> _IOC_SIZESHIFT) & _IOC_SIZEMASK)

#define IOC_IN            (_IOC_WRITE << _IOC_DIRSHIFT)
#define IOC_OUT         (_IOC_READ << _IOC_DIRSHIFT)
#define IOC_INOUT     ((_IOC_WRITE|_IOC_READ) << _IOC_DIRSHIFT)
#define IOCSIZE_MASK      (_IOC_SIZEMASK << _IOC_SIZESHIFT)
#define IOCSIZE_SHIFT      (_IOC_SIZESHIFT)
```

注意cmd不能为2。