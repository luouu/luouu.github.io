<!--
 * @Description: 轮询
 * @Author: luo_u
 * @Date: 2020-06-01 10:57:30
 * @LastEditTime: 2020-10-02 14:52:01
--> 

[TOC]

## poll

使用非阻塞 I/O的应用程序通常会使用 select()和 poll()查询是否可对设备进行无阻塞的访问。应用层调用select()最终会引发设备驱动中的 poll()函数被执行。

```c
unsigned int (*poll) (struct file *, struct poll_table *);

#define POLLIN		0x0001
#define POLLPRI		0x0002
#define POLLOUT		0x0004
#define POLLERR		0x0008
#define POLLHUP		0x0010
#define POLLNVAL	0x0020
#define POLLRDNORM	0x0040
#define POLLRDBAND	0x0080
#define POLLWRNORM	0x0100
#define POLLWRBAND	0x0200
#define POLLMSG		0x0400
#define POLLREMOVE	0x1000
#define POLLRDHUP       0x2000

void poll_wait(struct file * filp, wait_queue_head_t * wait_address, poll_table *p);
```
驱动要实现poll函数，主要工作为：对可能引起设备文件状态变化的等待队列调用 poll_wait()函数，将对应的等待队列头添加到 poll_table中，poll_wait()并不会引起阻塞。返回表示是否能对设备进行无阻塞读、写访问的掩码。
```c
static unsigned int xxx_poll(struct file *filp, struct poll_table_struct *wait)
{
	unsigned int mask = 0;
	struct xxx_object *dev = filp->private_data;


	poll_wait(filp, &dev->read_wait, wait);
	poll_wait(filp, &dev->write_wait, wait);


	if (...) //可读	
	{
		mask |= POLLIN | POLLRDNORM; /*数据可获得*/
	}


	if (...) //可写	
	{
		mask |= POLLOUT | POLLWRNORM; /*数据可写入*/
	}

	return mask;
}
```

```c
#include <sys/select.h>

/*清除一个文件描述符集*/
void FD_ZERO(fd_set *set);

/*将一个文件描述符从文件描述符集中清除*/
void FD_CLR(int fd, fd_set *set);

/*将一个文件描述符加入文件描述符集*/
void FD_SET(int fd, fd_set *set);

/*判断文件描述符是否被置位*/
int  FD_ISSET(int fd, fd_set *set);

int select(int nfds, fd_set *readfds, fd_set *writefds,
                  fd_set *exceptfds, struct timeval *timeout);
```
readfds、writefds、exceptfds 分别是被 select()监视的读、写和异常处理的文件描述符集合，numfds 参数是要检查的最高的文件描述符加1，timeout 参数是超时时间。
