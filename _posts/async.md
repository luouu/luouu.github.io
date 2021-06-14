<!--
 * @Description: 异步操作
 * @Author: luo_u
 * @Date: 2020-06-01 10:54:45
 * @LastEditTime: 2020-07-15 21:51:43
--> 

[TOC]

## 异步通知
异步通知是应用程序不需要一直查询设备状态，一旦设备就绪，则主动通知应用程序。为了使设备支持异步通知机制，驱动程序中涉及 3 项工作：

- 支持 F_SETOWN 命令，能在这个控制命令处理中设置 filp->f_owner 为对应进程 ID。不
过此项工作已由内核完成，设备驱动无需处理。

- 支持 F_SETFL 命令的处理，当应用改变 FASYNC 标志时，驱动程序中的fasync()函数将得以执行。

- 在设备资源可获得时，激发相应的信号。


![async](.img/async/async.png)

```c
int (*fasync) (int fd, struct file *filp, int mode);

/* 处理 FASYNC 标志变更 */
int fasync_helper(int fd, struct file *filp, int mode, struct fasync_struct **fa);

/* 释放信号 */
void kill_fasync(struct fasync_struct **fp, int sig, int band)
```

驱动中要实现fasync()函数，当应用改变 FASYNC 标志时，驱动程序中的fasync()函数将得以执行。
```c
static int xxx_fasync(int fd, struct file *filp, int mode)
{
	struct xxx_object *dev = filp->private_data;
	return fasync_helper(fd, filp, mode, &dev->async_queue);
}
```

在设备资源可以获得时，应该调用 kill_fasync()释放 SIGIO 信号，可读时band参数设置为 `POLL_IN`，可写时设置为 `POLL_OUT`。
```c
/* 产生异步读信号 */
if (dev->async_queue)
{
	kill_fasync(&dev->async_queue, SIGIO, POLL_IN);
}
```

最后在设备驱动的 release()函数中，应调用设备驱动的 fasync()函数将文件从异步通知的列表中删除。
```c
static int xxx_release(struct inode *inode, struct file *file)
{
	xxx_fasync(-1, file, 0);
	
	return 0;
}
```

## 异步IO