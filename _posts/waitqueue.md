<!--
 * @Description: 阻塞操作
 * @Author: luo_u
 * @Date: 2020-06-01 10:57:30
 * @LastEditTime: 2020-10-02 14:51:23
--> 

[TOC]

阻塞操作是指在执行设备操作时，若不能获得资源，则挂起的进程进入休眠状态，直到等待的条件被满足。而非阻塞操作并不挂起直接返回。

## waitqueue

阻塞进程可用等待队列来实现，等待队列可以用来同步对系统资源的访问。

```c
struct __wait_queue_head {
	spinlock_t lock;
	struct list_head task_list;
};
typedef struct __wait_queue_head wait_queue_head_t;

struct __wait_queue {
	unsigned int flags;
#define WQ_FLAG_EXCLUSIVE	0x01
	void *private;
	wait_queue_func_t func;
	struct list_head task_list;
};
typedef struct __wait_queue wait_queue_t;

/*初始化等待队列头*/
init_waitqueue_head(&queue);
#define DECLARE_WAIT_QUEUE_HEAD(name) \
	wait_queue_head_t name = __WAIT_QUEUE_HEAD_INITIALIZER(name)


/*定义并初始化一个名为 name 的等待队列*/
#define DECLARE_WAITQUEUE(name, tsk)					\
	wait_queue_t name = __WAITQUEUE_INITIALIZER(name, tsk)


/*添加/移除等待队列到等待队列头*/
void fastcall add_wait_queue(wait_queue_head_t *q, wait_queue_t *wait);
void fastcall remove_wait_queue(wait_queue_head_t *q, wait_queue_t *wait);

/*有条件等待*/
wait_event(queue, condition)
wait_event_interruptible(queue, condition)
wait_event_timeout(queue, condition, timeout)
wait_event_interruptible_timeout(queue, condition, timeout)

/*无条件等待 */
void __sched sleep_on(wait_queue_head_t *q);
void __sched interruptible_sleep_on(wait_queue_head_t *q);

/*唤醒队列*/
void wake_up(wait_queue_head_t *queue);
void wake_up_interruptible(wait_queue_head_t *queue);
```
sleep_on()函数将目前进程的状态置成 TASK_UNINTERRUPTIBLE，并定义一个等待队列，之后把它附属到等待队列头 q，直到资源可获得，q引导的等待队列被唤醒。interruptible_sleep_on()将进程的状态置成TASK_INTERRUPTIBLE，能被等待队列或者信号唤醒。

wake_up()应该与 wait_event()或 wait_event_timeout()成对使用，而 wake_up_interruptible()则应与 wait_event_interruptible()或 wait_event_interruptible_timeout()成对使用。 wake_up()可唤醒处于 TASK_INTERRUPTIBLE 和 TASK_UNINTERRUPTIBLE 的进程，而 wake_up_interruptible()只能唤醒处于 TASK_INTERRUPTIBLE 的进程。

```c
#define TASK_RUNNING		0
#define TASK_INTERRUPTIBLE	1
#define TASK_UNINTERRUPTIBLE	2

#define __set_current_state(state_value)			\
	do { current->state = (state_value); } while (0)
	
#define set_current_state(state_value)		\
	set_mb(current->state, (state_value))
```
内核中使用 set_current_state()函数或_ _add_current_state()函数来实现目前进程状态的改变，直接采用current->state = TASK_UNINTERRUPTIBLE类似的赋值语句也行。

如果是非阻塞访问，设备忙时，直接返回-EAGAIN。对于阻塞访问,会进行状态切换,通过schedule()调度其他进程执行。唤醒后，要通过 signal_pending(current) 判断是否是信号唤醒的，如果是立即返回-ERESTARTSYS。
```c
/* 改变进程状态为睡眠 */
__set_current_state(TASK_INTERRUPTIBLE);

/* 调度其他进程执行 */
schedule(); 	

if (signal_pending(current)) //如果是被信号唤醒
{
	ret = -ERESTARTSYS;
}
```
