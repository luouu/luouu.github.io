<!--
 * @Description: 
 * @Author: luo_u
 * @Date: 2020-07-26 08:59:17
 * @LastEditTime: 2020-07-26 09:09:21
--> 
/*
 * @Description: timer 延时
 * @Author: luo_u
 * @Date: 2020-07-15 22:10:20
 * @LastEditTime: 2020-07-15 22:11:27
 */


[TOC]

## timer

HZ 变量表示系统时钟频率，范围50-1200，默认1000。当时钟中断发生时，内核内部计数器的値就会加1，内部计数器由全局变量 jiffies 来表示。

```c
struct timer_list {
	struct list_head entry; /*定时器列表*/
	unsigned long expires; /*定时器到期时间*/
	void (*function)(unsigned long); /* 定时器处理函数*/
	unsigned long data; /*作为参数被传入定时器处理函数*/
	struct timer_base_s *base;
	int slack;
};


/*初始化定时器*/
void init_timer(struct timer_list * timer);

#define TIMER_INITIALIZER(_function, _expires, _data) {		\
		.entry = { .prev = TIMER_ENTRY_STATIC },	\
		.function = (_function),			\
		.expires = (_expires),				\
		.data = (_data),				\
		.base = &boot_tvec_bases,			\
		.slack = -1,					\
		__TIMER_LOCKDEP_MAP_INITIALIZER(		\
			__FILE__ ":" __stringify(__LINE__))	\
	}


#define DEFINE_TIMER(_name, _function, _expires, _data)		\
	struct timer_list _name =				\
		TIMER_INITIALIZER(_function, _expires, _data)


/*注册内核定时器，将定时器加入到内核动态定时器链表中*/
void add_timer(struct timer_list * timer);

/*删除定时器*/
int del_timer(struct timer_list * timer);

/*重装载定时器*/
int mod_timer(struct timer_list *timer, unsigned long expires);
```
内核定时器注册的处理函数只会执行一次，所以执行完后要重装载定时器，使其循环执行。


定时器的到期时间是在目前 jiffies 的基础是加一个时延，若为Hz，则表示延迟 1s。

```c
unsigned long msecs_to_jiffies(const unsigned int ms);
unsigned long usecs_to_jiffies(const unsigned int us);
unsigned long timespec_to_jiffies(const struct timespec *value);
```



## 内核延时

```c
void ndelay(unsigned long nsecs);
void udelay(unsigned long usecs);
void mdelay(unsigned long msecs);
```

这三种延时是根据CPU频率进行一定次数循环的忙等待。

```c
void msleep(unsigned int millisecs);
unsigned long msleep_interruptible(unsigned int millisecs);
void ssleep(unsigned int seconds);
```

进程睡眠指定的时间，其中msleep()、ssleep()不能被打断。msleep_interruptible()可被打断。

