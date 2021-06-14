<!--
 * @Description: 中断与时钟
 * @Author: luo_u
 * @Date: 2020-06-01 11:02:39
 * @LastEditTime: 2020-09-19 13:06:34
--> 

[TOC]

## 中断
根据中断入口跳转方法的不同，分为向量中断和非向量中断。采用向量中断的 CPU 通常为不同的中断分配不同的中断号，当检测到某中断号的中断到来后，就自动跳转到与该中断号对应的地址执行。不同中断号的中断有不同的入口地址。非向量中断的多个中断共享一个入口地址，进入该入口地址后再通过软件判断中断标志来识别具体是哪个中断。

中断处理程序是在中断上下文中运行的，它受到某些限制:
1. 不能向用户空间发送或接受数据
2. 不能使用可能引起阻塞的函数
3. 不能使用可能引起调度的函数

![](.img/irq/irq.png)

Linux 将中断处理程序分为顶半部(top half)和底半部(bottom half)。顶半部完成尽可能少的比较紧急的功能，往只是简单地读取寄存器中的中断状态并清除中断标志，这样才能服务更多的中断请求。底半部处理比较耗时的事情，而且可以被新的中断打断。Linux 实现底半部的机制主要有 tasklet、工作队列和软中断。

tasklet和内核定时器都是依靠软中断实现。软中断和tasklet 运行于软中断上下文，仍然属于原子上下文的一种，而工作队列则运行于进
程上下文。因此，软中断和 tasklet 处理函数中不能睡眠，而工作队列处理函数中允许睡眠。

`/proc/interrupts`文件可以获得系统中中断的统计信息，第1列是中断号，第2列是向对应CPU产后的中断的次数。

```c
#include <linux/interrupt.h> 
#include <linux/irq.h>

typedef irqreturn_t (*irq_handler_t)(int, void *);

int request_irq(unsigned int irq, irq_handler_t handler,
		unsigned long flags, const char *devname, void *dev_id);

void free_irq(unsigned int irq, void *dev_id);
```
irq 是要申请的硬件中断号。handler 是向系统登记的中断处理函数，是一个回调函数，中断发生时，系统调用这个函数，dev_id 可在共享中断中作为参数传入中断处理函数，也可以为 NULL。

flags 是中断处理的属性,可以指定中断的触发方式以及处理方式。
- IRQF_TRIGGER_RISING
- IRQF_TRIGGER_FALLING
- IRQF_TRIGGER_HIGH
- IRQF_TRIGGER_LOW 
- IRQF_ISABLED  表明中断处理程序是快速处理程序，快速处理程序被调用时屏蔽所有中断，慢速处理程序则不会屏蔽其他设备的驱动。
- IRQF_SHARED  表示多个设备共享中断，主要是为PCI设备服务。

request_irq()返回 0 表示成功；返回-EINVAL 表示中断号无效或处理函数指针为 NULL；返回-EBUSY 表示中断已经被占用且不能共享。

request_irq()相对应的释放中断的函数为 free_irq()。

共享中断到来时，会遍历执行共享此中断的所有中断处理程序，直到某一个函数返回 IRQ_HANDLED。在中断处理程序顶半部中，应对照传入的 dev_id 参数判断是否是本设备的中断，若不是返回 IRQ_NONE。

```c
void disable_irq(unsigned int irq);
void disable_irq_nosync(unsigned int irq);
void enable_irq(unsigned int irq);
```
这些函数用于使能和屏蔽一个中断源，disable_irq_nosync()立即返回，而disable_rq()会等待指定的中断被处理完，因此如果在顶半部调用，会引起系统的死锁。



## tasklet
tasklet不能休眠，同一个tasklet不能在两个CPU上同时运行。定义 tasklet 及其中断底半部处理函数并将两者关联，在中断顶半部中调度task执行。

```c
#include <linux/interrupt.h> 

struct tasklet_struct
{
	struct tasklet_struct *next;
	unsigned long state;
	atomic_t count;
	void (*func)(unsigned long);
	unsigned long data;
};

/*初始化tasklet，指定其处理函数及其传递的参数*/
void tasklet_init(struct tasklet_struct *t,
		  void (*func)(unsigned long), unsigned long data);

/*调度task执行*/
void tasklet_schedule(struct tasklet_struct *t);

/*销毁tasklet*/
void tasklet_kill(struct tasklet_struct *t);
```



## 工作队列
定义一个工作队列和一个底半部执行函数，并将其绑定，在中断顶半部调度工作队列执行。

```c
#include <linux/workqueue.h>

struct work_struct {
	atomic_long_t data;
	struct list_head entry;
	work_func_t func;
#ifdef CONFIG_LOCKDEP
	struct lockdep_map lockdep_map;
#endif
};


struct workqueue_struct {
	unsigned int		flags;		/* I: WQ_* flags */
	union {
		struct cpu_workqueue_struct __percpu	*pcpu;
		struct cpu_workqueue_struct		*single;
		unsigned long				v;
	} cpu_wq;				/* I: cwq's */
	struct list_head	list;		/* W: list of all workqueues */

	struct mutex		flush_mutex;	/* protects wq flushing */
	int			work_color;	/* F: current work color */
	int			flush_color;	/* F: current flush color */
	atomic_t		nr_cwqs_to_flush; /* flush in progress */
	struct wq_flusher	*first_flusher;	/* F: first flusher */
	struct list_head	flusher_queue;	/* F: flush waiters */
	struct list_head	flusher_overflow; /* F: flush overflow list */

	mayday_mask_t		mayday_mask;	/* cpus requesting rescue */
	struct worker		*rescuer;	/* I: rescue worker */

	int			saved_max_active; /* W: saved cwq max_active */
	const char		*name;		/* I: workqueue name */
#ifdef CONFIG_LOCKDEP
	struct lockdep_map	lockdep_map;
#endif
};


typedef void (*work_func_t)(struct work_struct *work);


/*初始化工作并将其与处理函数绑定*/
INIT_WORK(struct work_struct *work, work_func_t func)

/*调度工作执行*/
int schedule_work(struct work_struct *work);

bool flush_work(struct work_struct *work);
bool flush_work_sync(struct work_struct *work);
bool cancel_work_sync(struct work_struct *work);


#define create_workqueue(name)					
#define create_freezable_workqueue(name)			
#define create_singlethread_workqueue(name)			

void flush_workqueue(struct workqueue_struct *wq);

/*销毁工作队列*/
void destroy_workqueue(struct workqueue_struct *wq);
```



## delayed_work

```c
struct work_struct {
	atomic_long_t data;
	struct list_head entry;
	work_func_t func;
#ifdef CONFIG_LOCKDEP
	struct lockdep_map lockdep_map;
#endif
};


struct delayed_work {
	struct work_struct work;
	struct timer_list timer;
};

typedef void (*work_func_t)(struct work_struct *work);

INIT_DELAYED_WORK(struct delayed_work *dwork, work_func_t func)

/*调度delayed_work 在指定延时后执行*/
int schedule_delayed_work(struct delayed_work *dwork, unsigned long delay);


bool flush_delayed_work(struct delayed_work *dwork);
bool flush_delayed_work_sync(struct delayed_work *work);
bool cancel_delayed_work(struct delayed_work *dwork);
bool cancel_delayed_work_sync(struct delayed_work *dwork);
```



