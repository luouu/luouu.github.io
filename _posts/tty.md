<!--
 * @Description: 
 * @Author: luo_u
 * @Date: 2020-08-09 16:56:18
 * @LastEditTime: 2020-09-26 19:59:03
-->

[TOC]

## 终端

终端是一种字符型设备，它有多种类型，通常使用 tty 来简称各种类型的终端设备。 tty 是 Teletype 的缩写， Teletype 是最早出现的一种终端设备，很像电传打字机，是由 Teletype公司生产的。Linux 中包含如下几类终端设备。

- **串行端口终端(/dev/ttySn)**

串行端口终端(Serial Port Terminal)是使用计算机串行端口连接的终端设备。计算机把每个串行端口都看作是一个字符设备。这些串行端口所对应的设备名称是/dev/ttyS0(或/dev/tts/0)、/dev/ttyS1(或/dev/tts/1)等，设备号是4。在命令行上把标准输出重定向到端口对应的设备文件名上就可以通过该端口发送数据，例如，在命令行提示符下键入`echo test > /dev/ttyS1`会把单词“test”发送到连接在 ttyS1 端口的设备上。

USB-串口转换器对应设备结点通常为/dev/ttyUSB0， /dev/ttyUSB1 等。



- **伪终端(/dev/pty/)**

伪终端 pty(Pseudo Terminal)是成对的逻辑终端设备，并存在成对的设备文件。伪终端由pts(pseudo-terminal slave)和ptm(pseudo-terminal master)两部分组成。

pts伪造出一个标准的TTY设备，应用程序可以直接访问。应用程序向pts写入的数据，会直接反映到ptm上，同样，应用程序从pts读数据，则相当于直接从ptm读取。

而pym，根据具体情况具体实现。例如：要通过网络接口和终端设备交互，则pym需要打开对应的socket，将pts写来的数据，从socket送出，将从socket读取的数据，送回给pts。打开/dev/ptmx文件的时候，系统会自动在/dev/pts目录下创建一个新的设备文件。只要不关闭/dev/ptmx文件描述符，那么这个设备文件就会存在，一旦关闭，这个设备文件会自动消失。

在telnet，ssh等远程终端工具中会使用到伪终端，telnet通过网络协议与linux主机上的telnetd进程通讯，telnetd进程收到网络中的数据后，将数据写入/dev/ptmx，/dev/ptmx像管道一样将数据传递给/dev/pts/x，getty进程从pts/x读取数据传递给shell去执行。




- **控制终端(/dev/tty)**

如果当前进程有控制终端(Controlling Terminal)的话，那么/dev/tty 就是当前进程的控制终端的设备特殊文件。可以使用命令`ps –ax`来查看进程与哪个控制终端相连，使用命令`tty`可以查看它具体对应哪个实际终端设备。


- **控制台终端(/dev/ttyn，/dev/console)**

在 UNIX 系统中，计算机显示器通常被称为控制台终端(console)。当用户在控制台上登录时，使用的是 tty1。可以使用 Alt+[F1~F6]组合键切换到 tty2~tty6。tty1~tty6 等称为虚拟终端，tty0 则是当前所使用虚拟终端的一个别名，系统所产生的信息会发送到该终端上。用户登录不同的虚拟终端，就有多个不同的会话期存在。只有超级用户可以向/dev/tty0 进行写操作。

可以在系统启动命令行里指定kernel使用那个控制台终端，格式如下:
>console=device， options

- device 指代的是终端设备，可以是 tty0(前台的虚拟终端)、ttyx(第x个虚拟终端)、ttySx(第x个串口)、lp0(第一个并口)等。
- options 指代对 device 进行的设置，它取决于具体的设备驱动。对于串口设备，参数定义为:波特率、校验位(n/o/e)、位数，默认 options 是 9600n8。

用户可以在内核命令行中同时设定多个console，这样输出将会在所有的console上显示，而当用户调用open()打开/dev/console时，打开的是最后一个console。

通过查看 /proc/tty/drivers 文件可以获知tty设备驱动的信息。




## tty框架

tty可以分为如下几层：
- **核心层（tty core）**: 是tty设备的抽象
- **线路规程（tty line discipline）**: 是对上层和底层之间数据传输的协议转换, 不同类型的终端设备数据转换协议不同
- **驱动层（tty driver）**: 面向底层硬件的设备驱动

![](.img/tty/tty-arch.jpg)

![](.img/tty/tty-dev.bmp)

![](.img/tty/tty-ops.bmp)


```
├── hvc
├── ipwireless
├── serdev
├── vt
├── serial
│   ├── altera_uart.c
│   ├── amba-pl010.c
│   ├── amba-pl011.c
│   ├── amba-pl011.h
│   ├── atmel_serial.c
│   ├── atmel_serial.h
│   ├── rda-uart.c
│   ├── samsung.c
│   ├── samsung.h
│   ├── serial_core.c
│   ├── stm32-usart.c
│   ├── stm32-usart.h
├── tty_audit.c
├── tty_baudrate.c
├── tty_buffer.c
├── tty_io.c        #ttty设备创建和相关文件操作API
├── tty_ioctl.c
├── tty_jobctrl.c
├── tty_ldisc.c    #线路规程
├── tty_ldsem.c
├── tty_mutex.c
├── tty_port.c
├── n_tty.c
├── n_gsm.c
└── pty.c
```



## tty核心层

### tty_struct
```
struct tty_struct {
  int magic;
  struct kref kref;
  struct device *dev;
  struct tty_driver *driver;
  const struct tty_operations *ops;
  int index;

  /* Protects ldisc changes: Lock tty not pty */
  struct ld_semaphore ldisc_sem;
  struct tty_ldisc *ldisc;

  struct mutex atomic_write_lock;
  struct mutex legacy_mutex;
  struct mutex throttle_mutex;
  struct rw_semaphore termios_rwsem;
  struct mutex winsize_mutex;
  spinlock_t ctrl_lock;
  spinlock_t flow_lock;
  /* Termios values are protected by the termios rwsem */
  struct ktermios termios, termios_locked;
  struct termiox *termiox;  /* May be NULL for unsupported */
  char name[64];
  struct pid *pgrp;   /* Protected by ctrl lock */
  struct pid *session;
  unsigned long flags;
  int count;
  struct winsize winsize;   /* winsize_mutex */
  unsigned long stopped:1,  /* flow_lock */
          flow_stopped:1,
          unused:BITS_PER_LONG - 2;
  int hw_stopped;
  unsigned long ctrl_status:8,  /* ctrl_lock */
          packet:1,
          unused_ctrl:BITS_PER_LONG - 9;
  unsigned int receive_room;  /* Bytes free for queue */
  int flow_change;

  struct tty_struct *link;
  struct fasync_struct *fasync;
  wait_queue_head_t write_wait;
  wait_queue_head_t read_wait;
  struct work_struct hangup_work;
  void *disc_data;
  void *driver_data;
  spinlock_t files_lock;    /* protects tty_files list */
  struct list_head tty_files;

#define N_TTY_BUF_SIZE 4096

  int closing;
  unsigned char *write_buf;
  int write_cnt;
  /* If the tty has a pending do_SAK, queue it here - akpm */
  struct work_struct SAK_work;
  struct tty_port *port;
};
```



### tty_driver
```c
struct tty_driver
{        
  int magic;    /* magic number for this structure */
  struct kref kref; /* Reference management */
  struct cdev **cdevs;
  struct module *owner;
  const char  *driver_name;
  const char  *name;
  int name_base;  /* offset of printed name */
  int major;    /* major device number */
  int minor_start;  /* start of minor device number */
  unsigned int  num;  /* number of devices allocated */
  short type;   /* type of tty driver */
  short subtype;  /* subtype of tty driver */
  struct ktermios init_termios; /* Initial termios */
  unsigned long flags;    /* tty driver flags */
  struct proc_dir_entry *proc_entry; /* /proc fs entry */
  struct tty_driver *other; /* only used for the PTY driver */

  struct tty_struct **ttys;
  struct tty_port **ports;
  struct ktermios **termios;
  void *driver_state;

  const struct tty_operations *ops;
  struct list_head tty_drivers;
};
```


### tty_port
```c
struct tty_port {
	struct tty_bufhead	buf;		/* Locked internally */
	struct tty_struct	*tty;		/* Back pointer */
	struct tty_struct	*itty;		/* internal back ptr */
	const struct tty_port_operations *ops;	/* Port operations */
	const struct tty_port_client_operations *client_ops; /* Port client operations */
	spinlock_t		lock;		/* Lock protecting tty field */
	int			blocked_open;	/* Waiting to open */
	int			count;		/* Usage count */
	wait_queue_head_t	open_wait;	/* Open waiters */
	wait_queue_head_t	delta_msr_wait;	/* Modem status change */
	unsigned long		flags;		/* User TTY flags ASYNC_ */
	unsigned long		iflags;		/* Internal flags TTY_PORT_ */
	unsigned char		console:1,	/* port is a console */
				low_latency:1;	/* optional: tune for latency */
	struct mutex		mutex;		/* Locking */
	struct mutex		buf_mutex;	/* Buffer alloc lock */
	unsigned char		*xmit_buf;	/* Optional buffer */
	unsigned int		close_delay;	/* Close port delay */
	unsigned int		closing_wait;	/* Delay for output */
	int			drain_delay;	/* Set to zero if no pure time
						   based drain is needed else
						   set to size of fifo */
	struct kref		kref;		/* Ref counter */
	void 			*client_data;
};
```


### tty_port_operations
```c
struct tty_port_operations {
	int (*carrier_raised)(struct tty_port *port);
	void (*dtr_rts)(struct tty_port *port, int raise);
	void (*shutdown)(struct tty_port *port);
	int (*activate)(struct tty_port *port, struct tty_struct *tty);
	void (*destruct)(struct tty_port *port);
};
```


### tty_operations
```c
struct tty_operations {
  struct tty_struct * (*lookup)(struct tty_driver *driver,
      struct file *filp, int idx);
  int  (*install)(struct tty_driver *driver, struct tty_struct *tty);
  void (*remove)(struct tty_driver *driver, struct tty_struct *tty);
  int  (*open)(struct tty_struct * tty, struct file * filp);
  void (*close)(struct tty_struct * tty, struct file * filp);
  void (*shutdown)(struct tty_struct *tty);
  void (*cleanup)(struct tty_struct *tty);
  int  (*write)(struct tty_struct * tty,
          const unsigned char *buf, int count);
  int  (*put_char)(struct tty_struct *tty, unsigned char ch);
  void (*flush_chars)(struct tty_struct *tty);
  int  (*write_room)(struct tty_struct *tty);
  int  (*chars_in_buffer)(struct tty_struct *tty);
  int  (*ioctl)(struct tty_struct *tty,
        unsigned int cmd, unsigned long arg);
  long (*compat_ioctl)(struct tty_struct *tty,
           unsigned int cmd, unsigned long arg);
  void (*set_termios)(struct tty_struct *tty, struct ktermios * old);
  void (*throttle)(struct tty_struct * tty);
  void (*unthrottle)(struct tty_struct * tty);
  void (*stop)(struct tty_struct *tty);
  void (*start)(struct tty_struct *tty);
  void (*hangup)(struct tty_struct *tty);
  int (*break_ctl)(struct tty_struct *tty, int state);
  void (*flush_buffer)(struct tty_struct *tty);
  void (*set_ldisc)(struct tty_struct *tty);
  void (*wait_until_sent)(struct tty_struct *tty, int timeout);
  void (*send_xchar)(struct tty_struct *tty, char ch);
  int (*tiocmget)(struct tty_struct *tty);
  int (*tiocmset)(struct tty_struct *tty,
      unsigned int set, unsigned int clear);
  int (*resize)(struct tty_struct *tty, struct winsize *ws);
  int (*set_termiox)(struct tty_struct *tty, struct termiox *tnew);
  int (*get_icount)(struct tty_struct *tty,
        struct serial_icounter_struct *icount);
  int  (*get_serial)(struct tty_struct *tty, struct serial_struct *p);
  int  (*set_serial)(struct tty_struct *tty, struct serial_struct *p);
  void (*show_fdinfo)(struct tty_struct *tty, struct seq_file *m);
#ifdef CONFIG_CONSOLE_POLL
  int (*poll_init)(struct tty_driver *driver, int line, char *options);
  int (*poll_get_char)(struct tty_driver *driver, int line);
  void (*poll_put_char)(struct tty_driver *driver, int line, char ch);
#endif
  int (*proc_show)(struct seq_file *, void *);
};
```


### tty_register_driver()
```c
int tty_register_driver(struct tty_driver *driver)
{
  int error;
  int i;
  dev_t dev;
  struct device *d;

  /* 注册字符设备 */

  if (!driver->major) {
    error = alloc_chrdev_region(&dev, driver->minor_start,
            driver->num, driver->name);
    if (!error) {
      driver->major = MAJOR(dev);
      driver->minor_start = MINOR(dev);
    }
  } else {
    dev = MKDEV(driver->major, driver->minor_start);
    error = register_chrdev_region(dev, driver->num, driver->name);
  }

  if (driver->flags & TTY_DRIVER_DYNAMIC_ALLOC) {
    error = tty_cdev_add(driver, dev, 0, driver->num);
  }

  mutex_lock(&tty_mutex);
  list_add(&driver->tty_drivers, &tty_drivers);  //添加到全局链表 tty_drivers
  mutex_unlock(&tty_mutex);

  if (!(driver->flags & TTY_DRIVER_DYNAMIC_DEV)) {
    for (i = 0; i < driver->num; i++) {
      d = tty_register_device(driver, i, NULL); //注册tty设备
      if (IS_ERR(d)) {
        error = PTR_ERR(d);
        goto err_unreg_devs;
      }
    }
  }
  proc_tty_register_driver(driver);  //添加 proc 文件系统
  driver->flags |= TTY_DRIVER_INSTALLED;
  return 0;
}   

static int tty_cdev_add(struct tty_driver *driver, dev_t dev,
    unsigned int index, unsigned int count)
{
  int err;

  driver->cdevs[index] = cdev_alloc();
  if (!driver->cdevs[index])
    return -ENOMEM;
  driver->cdevs[index]->ops = &tty_fops;  //设置文件操作函数集 tty_fops
  driver->cdevs[index]->owner = driver->owner;
  err = cdev_add(driver->cdevs[index], dev, count);
  if (err)
    kobject_put(&driver->cdevs[index]->kobj);
  return err;
}
```
注册字符设备，将该 tty_driver->tty_drivers 添加到全局链表 tty_drivers。添加 proc 文件系统。设置文件操作函数集 tty_fops。



### tty_open()

```c
static int tty_open(struct inode *inode, struct file *filp)
{
  struct tty_struct *tty;
  int noctty, retval;
  dev_t device = inode->i_rdev;
  unsigned saved_flags = filp->f_flags;

  tty = tty_open_current_tty(device, filp); /* 获取当前线程锁定的tty */
  if (!tty)
    tty = tty_open_by_driver(device, inode, filp); /* 通过查找tty驱动打开tty设备 */

  if (tty->ops->open)
    retval = tty->ops->open(tty, filp);
}
```

```c
static struct tty_struct *tty_open_current_tty(dev_t device, struct file *filp)
{
	struct tty_struct *tty;

	/* 如果当前设备是/dev/tty则尝试重新打开 */
	if (device != MKDEV(TTYAUX_MAJOR, 0))
		return NULL;

	tty = get_current_tty();
	retval = tty_reopen(tty);

	return tty;
}

static int tty_reopen(struct tty_struct *tty)
{
	struct tty_driver *driver = tty->driver;
	struct tty_ldisc *ld;

	ld = tty_ldisc_ref_wait(tty);
	if (ld) {
		tty_ldisc_deref(ld);
	} else {
		retval = tty_ldisc_lock(tty, 5 * HZ);

		if (!tty->ldisc)
			retval = tty_ldisc_reinit(tty, tty->termios.c_line);  
		tty_ldisc_unlock(tty);
	}
}
```
调用线路规程的[tty_ldisc_reinit()](#tty_ldisc_reinit())函数重新初始化tty设备的线路规程。



```c
static struct tty_struct *tty_open_by_driver(dev_t device, struct inode *inode,
               struct file *filp)
{
	struct tty_struct *tty;
	struct tty_driver *driver = NULL;
	int index = -1;

	/* 从 tty_drivers 全局链表中获取注册的 tty_driver */
	driver = tty_lookup_driver(device, filp, &index);

	通过 tty_drivers 查找 tty_struct
	tty = tty_driver_lookup_tty(driver, filp, index);

	if (tty) {
		tty_reopen(tty);
	} else { 
		tty = tty_init_dev(driver, index);
	}

	return tty;
}

struct tty_struct *tty_init_dev(struct tty_driver *driver, int idx)
{
	struct tty_struct *tty;
	int retval;

	tty = alloc_tty_struct(driver, idx);

	if (!tty->port)
		tty->port = driver->ports[idx];

	tty_ldisc_lock(tty, 5 * HZ);
	tty->port->itty = tty;

	tty_ldisc_setup(tty, tty->link);
	return tty;
}

struct tty_struct *alloc_tty_struct(struct tty_driver *driver, int idx)
{
	struct tty_struct *tty;

	tty = kzalloc(sizeof(*tty), GFP_KERNEL);

	kref_init(&tty->kref);
	tty->magic = TTY_MAGIC;

	tty_ldisc_init(tty);

	tty->session = NULL;
	tty->pgrp = NULL;
	mutex_init(&tty->legacy_mutex);
	mutex_init(&tty->throttle_mutex);
	init_rwsem(&tty->termios_rwsem);
	mutex_init(&tty->winsize_mutex);
	init_ldsem(&tty->ldisc_sem);
	init_waitqueue_head(&tty->write_wait);
	init_waitqueue_head(&tty->read_wait);
	INIT_WORK(&tty->hangup_work, do_tty_hangup);
	mutex_init(&tty->atomic_write_lock);
	spin_lock_init(&tty->ctrl_lock);
	spin_lock_init(&tty->flow_lock);
	spin_lock_init(&tty->files_lock);
	INIT_LIST_HEAD(&tty->tty_files);
	INIT_WORK(&tty->SAK_work, do_SAK_work);

	tty->driver = driver;
	tty->ops = driver->ops;  /* 设置tty设备文件操作接口 */
	tty->index = idx;
	tty_line_name(driver, idx, tty->name);
	tty->dev = tty_get_device(tty);

	return tty;
}

int tty_ldisc_setup(struct tty_struct *tty, struct tty_struct *o_tty)
{
	tty_ldisc_open(tty, tty->ldisc);
	return 0;
}
```
实例化一个 tty_struct，调用[tty_ldisc_open()](#tty_ldisc_open())打开一个线路规程。最后调用 tty_struct->ops->open 函数，其实是tty_driver->ops->open，也就是uart_ops中的open函数。


```c
static int uart_open(struct tty_struct *tty, struct file *filp)
{
	struct uart_driver *drv = tty->driver->driver_state;
	int retval, line = tty->index;
	struct uart_state *state = drv->state + line;

	tty->driver_data = state;

	retval = tty_port_open(&state->port, tty, filp);

	return retval;
}

int tty_port_open(struct tty_port *port, struct tty_struct *tty,
							struct file *filp)
{
	spin_lock_irq(&port->lock);
	++port->count;
	spin_unlock_irq(&port->lock);
	tty_port_tty_set(port, tty);

	mutex_lock(&port->mutex);

	if (!tty_port_initialized(port)) {
		clear_bit(TTY_IO_ERROR, &tty->flags);
		if (port->ops->activate) {
			int retval = port->ops->activate(port, tty);
		}
		tty_port_set_initialized(port, 1);
	}
	mutex_unlock(&port->mutex);
	return tty_port_block_til_ready(port, tty, filp);
}


static int uart_port_activate(struct tty_port *port, struct tty_struct *tty)
{
	struct uart_state *state = container_of(port, struct uart_state, port);
	struct uart_port *uport;

	return uart_startup(tty, state, 0);
}

static int uart_startup(struct tty_struct *tty, struct uart_state *state,
		int init_hw)
{
	struct tty_port *port = &state->port;

	return uart_port_startup(tty, state, init_hw);
}

static int uart_port_startup(struct tty_struct *tty, struct uart_state *state,
		int init_hw)
{
	struct uart_port *uport = uart_port_check(state);
	unsigned long page;
	unsigned long flags = 0;
	int retval = 0;

	if (uport->type == PORT_UNKNOWN)
		return 1;

	uart_change_pm(state, UART_PM_STATE_ON);

	page = get_zeroed_page(GFP_KERNEL);

	uart_port_lock(state, flags);
	if (!state->xmit.buf) {
		state->xmit.buf = (unsigned char *) page;
		uart_circ_clear(&state->xmit);
		uart_port_unlock(uport, flags);
	} else {
		uart_port_unlock(uport, flags);
		free_page(page);
	}

	retval = uport->ops->startup(uport);
	if (retval == 0) {
		if (uart_console(uport) && uport->cons->cflag) {
			tty->termios.c_cflag = uport->cons->cflag;
			uport->cons->cflag = 0;
		}

		uart_change_speed(tty, state, NULL);

		if (init_hw && C_BAUD(tty))
			uart_port_dtr_rts(uport, 1);
	}
}
```
最终调用的是state->uart_port->ops->startup()函数。


### tty_write()
```c
static ssize_t tty_write(struct file *file, const char __user *buf,
						size_t count, loff_t *ppos)
{
 	struct tty_ldisc *ld;
	ssize_t ret;
	struct tty_struct *tty = file_tty(file);
	
	ld = tty->ldisc;

	if (!ld)
		return hung_up_tty_write(file, buf, count, ppos);
	if (!ld->ops->write)
		ret = -EIO;
	else
		ret = do_tty_write(ld->ops->write, tty, file, buf, count);
	tty_ldisc_deref(ld);
	return ret;
}

static inline ssize_t do_tty_write(
	ssize_t (*write)(struct tty_struct *, struct file *, const unsigned char *, size_t),
	struct tty_struct *tty,
	struct file *file,
	const char __user *buf,
	size_t count)
{
	ssize_t ret, written = 0;
	unsigned int chunk;

	ret = tty_write_lock(tty, file->f_flags & O_NDELAY);

	chunk = 2048;
	if (test_bit(TTY_NO_WRITE_SPLIT, &tty->flags))
		chunk = 65536;
	if (count < chunk)
		chunk = count;

	if (tty->write_cnt < chunk) {
		unsigned char *buf_chunk;

		if (chunk < 1024)
			chunk = 1024;

		buf_chunk = kmalloc(chunk, GFP_KERNEL);
		if (!buf_chunk) {
			ret = -ENOMEM;
			goto out;
		}
		kfree(tty->write_buf);
		tty->write_cnt = chunk;
		tty->write_buf = buf_chunk;
	}

	for (;;) {
		size_t size = count;
		if (size > chunk)
			size = chunk;
		ret = -EFAULT;
		if (copy_from_user(tty->write_buf, buf, size))
			break;
		ret = write(tty, file, tty->write_buf, size);
		if (ret <= 0)
			break;
		written += ret;
		buf += ret;
		count -= ret;
		if (!count)
			break;
		ret = -ERESTARTSYS;
		if (signal_pending(current))
			break;
		cond_resched();
	}
	if (written) {
		tty_update_time(&file_inode(file)->i_mtime);
		ret = written;
	}
out:
	tty_write_unlock(tty);
	return ret;
}
```


## 串口驱动层

### uart_driver

uart_driver描述串口驱动，包含了串口设备名、串口驱动名、主次设备号、串口控制台等信息。
```c
struct uart_driver {
	struct module   *owner;
	const char    *driver_name;
	const char    *dev_name;
	int      major;
	int      minor;
	int      nr;
	struct console    *cons;
	struct uart_state *state;
	struct tty_driver *tty_driver;
};


struct uart_state {
	struct tty_port		port;
	int			pm_state;
	struct circ_buf		xmit;
	struct uart_port	*uart_port;	// 对应于一个串口设备
};   
```



### uart_port

uart_port 用于描述一个UART端口的I/O端口或I/O内存地址、FIFO大小、端口类型等信息。

```c
struct uart_port {　
	spinlock_t    lock;     /* port lock */
	unsigned long   iobase;     /* in/out[bwl] */
	unsigned char __iomem *membase;   /* read/write[bwl] */
	unsigned int    (*serial_in)(struct uart_port *, int);
	void      (*serial_out)(struct uart_port *, int, int);
	void      (*set_termios)(struct uart_port *,
						struct ktermios *new,
						struct ktermios *old);
	void      (*set_ldisc)(struct uart_port *,
				struct ktermios *);
	unsigned int    (*get_mctrl)(struct uart_port *);
	void      (*set_mctrl)(struct uart_port *, unsigned int);
	unsigned int    (*get_divisor)(struct uart_port *,
					unsigned int baud,
					unsigned int *frac);
	void      (*set_divisor)(struct uart_port *,
					unsigned int baud,
					unsigned int quot,
					unsigned int quot_frac);
	int     (*startup)(struct uart_port *port);
	void      (*shutdown)(struct uart_port *port);
	void      (*throttle)(struct uart_port *port);
	void      (*unthrottle)(struct uart_port *port);
	int     (*handle_irq)(struct uart_port *);
	void      (*pm)(struct uart_port *, unsigned int state,
				unsigned int old);
	void      (*handle_break)(struct uart_port *);
	int     (*rs485_config)(struct uart_port *,
			struct serial_rs485 *rs485);
	int     (*iso7816_config)(struct uart_port *,
				struct serial_iso7816 *iso7816);
	unsigned int    irq;      /* irq number */
	unsigned long   irqflags;   /* irq flags  */
	unsigned int    uartclk;    /* base uart clock */
	unsigned int    fifosize;   /* tx fifo size */
	unsigned char   x_char;     /* xon/xoff char */
	unsigned char   regshift;   /* reg offset shift */
	unsigned char   iotype;     /* io access style */
	unsigned char   quirks;     /* internal quirks */
	unsigned int    read_status_mask; /* driver specific */
	unsigned int    ignore_status_mask; /* driver specific */
	struct uart_state *state;     /* pointer to parent state */
	struct uart_icount  icount;     /* statistics */
	struct console    *cons;      /* struct console, if any */
	upf_t     flags;
	upstat_t    status; 
	int     hw_stopped;   /* sw-assisted CTS flow state */
	unsigned int    mctrl;      /* current modem ctrl settings */
	unsigned int    timeout;    /* character-based timeout */
	unsigned int    type;     /* port type */
	const struct uart_ops *ops;
	unsigned int    custom_divisor;
	unsigned int    line;     /* port index */
	unsigned int    minor;
	resource_size_t   mapbase;    /* for ioremap */
	resource_size_t   mapsize;
	struct device   *dev;     /* parent device */
	unsigned char   hub6;     /* this should be in the 8250 driver */
	unsigned char   suspended;
	unsigned char   unused[2];
	const char    *name;      /* port name */
	struct attribute_group  *attr_group;    /* port specific attributes */
	const struct attribute_group **tty_groups;  /* all attributes (serial core use only) */
	struct serial_rs485     rs485;
	struct serial_iso7816 iso7816;                                                         void      *private_data;    /* generic platform data pointer */
};
```
uart_port对应一个串口设备，用于描述串口端口的I/O端口或I/O内存地址、FIFO大小、端口类型、串口时钟等信息，需要实现串口相关操作方法的 uart_ops 结构体。
```c
struct uart_ops {
	unsignedint(*tx_empty)(struct uart_port *);	/* 串口的Tx FIFO缓存是否为空 */
	void(*set_mctrl)(struct uart_port *,unsignedint mctrl);	/* 设置串口modem控制 */
	unsignedint(*get_mctrl)(struct uart_port *);	/* 获取串口modem控制 */
	void(*stop_tx)(struct uart_port *);	/* 禁止串口发送数据 */
	void(*start_tx)(struct uart_port *);	/* 使能串口发送数据 */
	void(*send_xchar)(struct uart_port *,char ch);	/* 发送xChar */
	void(*stop_rx)(struct uart_port *);	/* 禁止串口接收数据 */
	void(*enable_ms)(struct uart_port *);	/* 使能modem的状态信号 */
	void(*break_ctl)(struct uart_port *,int ctl);	/* 设置break信号 */
	int(*startup)(struct uart_port *);	/* 启动串口,应用程序打开串口设备文件时,该函数会被调用 */
	void(*shutdown)(struct uart_port *);	/* 关闭串口,应用程序关闭串口设备文件时,该函数会被调用 */
	void(*set_termios)(struct uart_port *,struct ktermios *new,struct ktermios *old);	/* 设置串口参数 */
	void(*pm)(struct uart_port *,unsignedint state,
	unsignedint oldstate);	/* 串口电源管理 */
	int(*set_wake)(struct uart_port *,unsignedint state);	/* */
	constchar*(*type)(struct uart_port *);	/* 返回一描述串口类型的字符串 */
	void(*release_port)(struct uart_port *);	/* 释放串口已申请的IO端口/IO内存资源,必要时还需iounmap */
	int(*request_port)(struct uart_port *);	/* 申请必要的IO端口/IO内存资源,必要时还可以重新映射串口端口 */
	void(*config_port)(struct uart_port *,int);	/* 执行串口所需的自动配置 */
	int(*verify_port)(struct uart_port *,struct serial_struct *);	/* 核实新串口的信息 */
	int(*ioctl)(struct uart_port *,unsignedint,unsignedlong);	/* IO控制 */
};
```



### uart_driver注册
```c
static struct uart_driver s3c24xx_uart_drv = {
	.owner		= THIS_MODULE,
	.driver_name	= "s3c2410_serial",
	.nr		= CONFIG_SERIAL_SAMSUNG_UARTS,
	.cons		= S3C24XX_SERIAL_CONSOLE,
	.dev_name	= S3C24XX_SERIAL_NAME,
	.major		= S3C24XX_SERIAL_MAJOR,
	.minor		= S3C24XX_SERIAL_MINOR,
};

static struct s3c24xx_uart_port
s3c24xx_serial_ports[CONFIG_SERIAL_SAMSUNG_UARTS] = {
	[0] = {
		.port = {
			.lock		= __PORT_LOCK_UNLOCKED(0),
			.iotype		= UPIO_MEM,
			.uartclk	= 0,
			.fifosize	= 16,
			.ops		= &s3c24xx_serial_ops,
			.flags		= UPF_BOOT_AUTOCONF,
			.line		= 0,
		}
	},
	[1] = {
		.port = {
			.lock		= __PORT_LOCK_UNLOCKED(1),
			.iotype		= UPIO_MEM,
			.uartclk	= 0,
			.fifosize	= 16,
			.ops		= &s3c24xx_serial_ops,
			.flags		= UPF_BOOT_AUTOCONF,
			.line		= 1,
		}
	},
};

static int s3c24xx_serial_probe(struct platform_device *pdev)
{
	struct device_node *np = pdev->dev.of_node;
	struct s3c24xx_uart_port *ourport;
	int index = probe_index;
	int ret;

	ourport = &s3c24xx_serial_ports[index];

	ourport->drv_data = s3c24xx_get_driver_data(pdev);

	ourport->baudclk = ERR_PTR(-EINVAL);
	ourport->info = ourport->drv_data->info;
	ourport->cfg = (dev_get_platdata(&pdev->dev)) ?
			dev_get_platdata(&pdev->dev) :
			ourport->drv_data->def_cfg;

	if (ourport->drv_data->fifosize[index])
		ourport->port.fifosize = ourport->drv_data->fifosize[index];
	else if (ourport->info->fifosize)
		ourport->port.fifosize = ourport->info->fifosize;

	ourport->min_dma_size = max_t(int, ourport->port.fifosize,
				    dma_get_cache_alignment());

	ret = s3c24xx_serial_init_port(ourport, pdev);

	if (!s3c24xx_uart_drv.state) {
		ret = uart_register_driver(&s3c24xx_uart_drv);
	}

  /* 将 uart_port 注册到 uart_driver */
	uart_add_one_port(&s3c24xx_uart_drv, &ourport->port);
	platform_set_drvdata(pdev, &ourport->port);

	clk_disable_unprepare(ourport->clk);

	ret = s3c24xx_serial_cpufreq_register(ourport);

	probe_index++;

	return 0;
}

static struct platform_driver samsung_serial_driver = {
	.probe		= s3c24xx_serial_probe,
	.remove		= s3c24xx_serial_remove,
	.id_table	= s3c24xx_serial_driver_ids,
	.driver		= {
		.name	= "samsung-uart",
		.pm	= SERIAL_SAMSUNG_PM_OPS,
		.of_match_table	= of_match_ptr(s3c24xx_uart_dt_match),
	},
};

module_platform_driver(samsung_serial_driver);
```
先实例化一个 uart_driver 结构体，再调用 [uart_register_driver()](#uart_register_driver())函数注册到内核。通过 [uart_add_one_port()](#uart_add_one_port())函数向该驱动添加uart_port。





### uart_register_driver()

```c
static const struct tty_port_operations uart_port_ops = {
	.carrier_raised = uart_carrier_raised,
	.dtr_rts	= uart_dtr_rts,
	.activate	= uart_port_activate,
	.shutdown	= uart_tty_port_shutdown,
};

int uart_register_driver(struct uart_driver *drv) 
{ 
	struct tty_driver *normal; 
	int i, retval; 

	BUG_ON(drv->state); 
	
	/*
	* Maybe we should be using a slab cache for this, especially if
	* we have a large number of ports to handle.
	*/
	drv->state = kzalloc(sizeof(struct uart_state) * drv->nr, GFP_KERNEL);
	if (!drv->state)
		goto out;
	
	normal = alloc_tty_driver(drv->nr);
	if (!normal)
		goto out_kfree;
	
	drv->tty_driver = normal;
	
	normal->driver_name = drv->driver_name;
	normal->name    = drv->dev_name;
	normal->major   = drv->major;
	normal->minor_start = drv->minor;
	normal->type    = TTY_DRIVER_TYPE_SERIAL;
	normal->subtype   = SERIAL_TYPE_NORMAL;
	normal->init_termios  = tty_std_termios;
	normal->init_termios.c_cflag = B9600 | CS8 | CREAD | HUPCL | CLOCAL;
	normal->init_termios.c_ispeed = normal->init_termios.c_ospeed = 9600;
	normal->flags   = TTY_DRIVER_REAL_RAW | TTY_DRIVER_DYNAMIC_DEV;
	normal->driver_state    = drv;
	tty_set_operations(normal, &uart_ops);  //将tty_driver的操作集设为uart_ops
	
	for (i = 0; i < drv->nr; i++) {
		struct uart_state *state = drv->state + i;
		struct tty_port *port = &state->port;
	
		tty_port_init(port);
		port->ops = &uart_port_ops;
		port->close_delay     = HZ / 2; /* .5 seconds */
		port->closing_wait    = 30 * HZ;/* 30 seconds */
	}
	
	retval = tty_register_driver(normal);
}
```
分配一个 tty_driver，对 tty_driver 进行设置，其中包括默认波特率、校验方式等，然后调用[tty_register_driver()](#tty_register_driver())函数注册 tty_driver。根据 uart_driver->nr 来申请 nr 个 uart_state 空间，用来存放驱动所支持的串口端口的物理信息，每一个 uart_state 都有一个 uart_port。



### uart_add_one_port()

```c
int uart_add_one_port(struct uart_driver *drv, struct uart_port *uport)
{
	struct uart_state *state;
	struct tty_port *port;
	int ret = 0;
	struct device *tty_dev;
	int num_groups;

	if (uport->line >= drv->nr)
		return -EINVAL;

	state = drv->state + uport->line;
	port = &state->port;

	mutex_lock(&port_mutex);
	mutex_lock(&port->mutex);

	atomic_set(&state->refcount, 1);
	init_waitqueue_head(&state->remove_wait);
	state->uart_port = uport;
	uport->state = state;

	state->pm_state = UART_PM_STATE_UNDEFINED;
	uport->cons = drv->cons;
	uport->minor = drv->tty_driver->minor_start + uport->line;
	uport->name = kasprintf(GFP_KERNEL, "%s%d", drv->dev_name,
				drv->tty_driver->name_base + uport->line);

	if (!(uart_console(uport) && (uport->cons->flags & CON_ENABLED))) {
		spin_lock_init(&uport->lock);
		lockdep_set_class(&uport->lock, &port_lock_key);
	}
	if (uport->cons && uport->dev)
		of_console_check(uport->dev->of_node, uport->cons->name, uport->line);

	uart_configure_port(drv, state, uport);

	port->console = uart_console(uport);

	num_groups = 2;
	if (uport->attr_group)
		num_groups++;

	uport->tty_groups = kcalloc(num_groups, sizeof(*uport->tty_groups),
				    GFP_KERNEL);
	uport->tty_groups[0] = &tty_dev_attr_group;
	if (uport->attr_group)
		uport->tty_groups[1] = uport->attr_group;

	tty_dev = tty_port_register_device_attr_serdev(port, drv->tty_driver,
			uport->line, uport->dev, port, uport->tty_groups);

	uport->flags &= ~UPF_DEAD;
}
```
將 uart_prot 关联到 uart_driver 对应的 state。




## 线路规程
```c
struct tty_ldisc {
	struct tty_ldisc_ops *ops;
	struct tty_struct *tty;
};

struct tty_ldisc_ops {
	int	magic;
	char	*name;
	int	num;
	int	flags;

	int	(*open)(struct tty_struct *);
	void	(*close)(struct tty_struct *);
	void	(*flush_buffer)(struct tty_struct *tty);
	ssize_t	(*read)(struct tty_struct *tty, struct file *file,
			unsigned char __user *buf, size_t nr);
	ssize_t	(*write)(struct tty_struct *tty, struct file *file,
			 const unsigned char *buf, size_t nr);
	int	(*ioctl)(struct tty_struct *tty, struct file *file,
			 unsigned int cmd, unsigned long arg);
	int	(*compat_ioctl)(struct tty_struct *tty, struct file *file,
				unsigned int cmd, unsigned long arg);
	void	(*set_termios)(struct tty_struct *tty, struct ktermios *old);
	__poll_t (*poll)(struct tty_struct *, struct file *,
			     struct poll_table_struct *);
	int	(*hangup)(struct tty_struct *tty);
	void	(*receive_buf)(struct tty_struct *, const unsigned char *cp,
			       char *fp, int count);
	void	(*write_wakeup)(struct tty_struct *);
	void	(*dcd_change)(struct tty_struct *, unsigned int);
	int	(*receive_buf2)(struct tty_struct *, const unsigned char *cp,
				char *fp, int count);

	struct  module *owner;
	int refcount;
};
```


### ldisc注册
```c
static struct tty_ldisc_ops n_tty_ops = {
	.magic           = TTY_LDISC_MAGIC,
	.name            = "n_tty",
	.open            = n_tty_open,
	.close           = n_tty_close,
	.flush_buffer    = n_tty_flush_buffer,
	.read            = n_tty_read,
	.write           = n_tty_write,
	.ioctl           = n_tty_ioctl,
	.set_termios     = n_tty_set_termios,
	.poll            = n_tty_poll,
	.receive_buf     = n_tty_receive_buf,
	.write_wakeup    = n_tty_write_wakeup,
	.receive_buf2	 = n_tty_receive_buf2,
};

void __init n_tty_init(void)
{
	tty_register_ldisc(N_TTY, &n_tty_ops);
}
```
实现代码在n_tty.c中，调用[tty_register_ldisc()](#tty_register_ldisc())函数，注册线路规程的操作函数集。


### tty_register_ldis()



### tty_ldisc_reinit()
```c
int tty_ldisc_reinit(struct tty_struct *tty, int disc)
{
	struct tty_ldisc *ld;

	ld = tty_ldisc_get(tty, disc);

	if (tty->ldisc) {
		tty_ldisc_close(tty, tty->ldisc);
		tty_ldisc_put(tty->ldisc);
	}

	tty->ldisc = ld;
	tty_set_termios_ldisc(tty, disc);
	retval = tty_ldisc_open(tty, tty->ldisc);
}
```
设置tty的线路规程，调用 [tty_ldisc_open()](#tty_ldisc_open()) 函数打开一个线路规程。



### tty_ldisc_get()
```c
static struct tty_ldisc *tty_ldisc_get(struct tty_struct *tty, int disc)
{
	struct tty_ldisc *ld;
	struct tty_ldisc_ops *ldops;

	ldops = get_ldops(disc);

	ld = kmalloc(sizeof(struct tty_ldisc), GFP_KERNEL | __GFP_NOFAIL);
	ld->ops = ldops;
	ld->tty = tty;

	return ld;
}
```

### tty_ldisc_open()
```c

static int tty_ldisc_open(struct tty_struct *tty, struct tty_ldisc *ld)
{
	WARN_ON(test_and_set_bit(TTY_LDISC_OPEN, &tty->flags));
	if (ld->ops->open) {
		ret = ld->ops->open(tty);
	}
	return 0;
}
```



## microcom

microcom工具可以用来调度串口，它会将stdin的字节复制到TTY，并从TTY复制到stdout。

```shell
microcom [-d DELAY] [-t TIMEOUT] [-s SPEED] [-X] TTY

选项：
    -d      Wait up to DELAY ms for TTY output before sending every
            next byte to it
    -t      Exit if both stdin and TTY are silent for TIMEOUT ms
    -s      Set serial line to SPEED
    -X      Disable special meaning of NUL and Ctrl-X from stdin
```

如果想回显输入的字符，可以使用以下命令：
>tee /dev/stderr | microcom /dev/ttyS1




## 参考

https://blog.csdn.net/lizuobin2/article/details/51773305

https://www.bbsmax.com/R/obzbBZNBzE/

https://www.shuzhiduo.com/A/ZOJP4pxOJv/



