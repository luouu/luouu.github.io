---
layout: post
title: 原子变量与自旋锁与信号量
categories: [dev]
description: 
keywords: 
topmost: false
---

* TOC
{:toc}

## 原子操作

原子操作是一种不可以被打断的操作。需要硬件的支持，

### 整形原子操作

```c
typedef struct {
	volatile int counter;
} atomic_t;

/* 定义原子变量, 并初始化 */
void atomic_set(atomic_t *v, int i);
atomic_t v = ATOMIC_INIT(0); 

atomic_read(atomic_t *v);  /* 获取原子变量的值 */

void atomic_add(int i, atomic_t *v); /* 原子变量增加 i */
void atomic_sub(int i, atomic_t *v); /* 原子变量减少 i */

/*原子变量自增/自减*/
void atomic_inc(atomic_t *v);
void atomic_dec(atomic_t *v);

/*原子变量执行自增、自减和减操作，测试其是否为0，为0返回true，否则返回 false*/
int atomic_add_return(int i, atomic_t *v);
int atomic_sub_return(int i, atomic_t *v);
int atomic_inc_return(atomic_t *v);
int atomic_dec_return(atomic_t *v);

/*原子变量进行加/减和自增/自减操作,并返回新的值*/
int atomic_add_return(int i, atomic_t *v);
int atomic_sub_return(int i, atomic_t *v);
int atomic_inc_return(atomic_t *v);
int atomic_dec_return(atomic_t *v);
```

### 位原子操作

原子位操作，可对数据的每一位进行操作。

```c
/*设置addr地址的第nr位*/
void set_bit(unsigned long nr, const volatil void *addr);  

/*清除addr地址的第nr位*/
void clear_bit(unsigned long nr, const volatil void *addr);  

/*对addr地址的第nr位进行反置*/
void change_bit(unsigned long nr, const volatil void *addr);  

/*返回addr地址的第nr位*/
int test_bit(unsigned long nr, const volatil void *addr);  

/*先返回addr地址的第nr位，再执行相应bit操作*/
int test_and_set_bit(unsigned long nr, const volatil void *addr);
int test_and_clear_bit(unsigned long nr, const volatil void *addr);
int test_and_change_bit(unsigned long nr, const volatil void *addr);
```


## 自旋锁
自旋锁(spin lock)是一种对临界资源进行互斥访问的手段。自旋锁主要针对SMP或单CPU但内核可抢占的情况。自旋锁可以保证临界区不受别的CPU和本CPU 内的抢占进程打扰，但受到中断和底半部的影响。

自旋锁是一种**忙等待**，如果得不到锁，则会自旋在那里，不会引起调用者睡眠。因此，只有在占用锁的时间极短的情况下，使用自旋锁才是合理的。自旋锁锁定期间不能调用可能引起进程调度的函数，也不能递归使用。只允许一个持有者。


```c
typedef struct spinlock {
	union {
		struct raw_spinlock rlock;

#ifdef CONFIG_DEBUG_LOCK_ALLOC
# define LOCK_PADSIZE (offsetof(struct raw_spinlock, dep_map))
		struct {
			u8 __padding[LOCK_PADSIZE];
			struct lockdep_map dep_map;
		};
#endif
	};
} spinlock_t;


/*初始化自旋锁*/
void spin_lock_init(spinlock_t *lock);

/*获得自旋锁*/
void spin_lock(spinlock_t *lock);

/*尝试获得自旋锁，如果能立即获得锁，它获得锁并返回真；否则立即返回假，不再自旋*/
int spin_trylock(spinlock_t *lock);

/*释放自旋锁*/
void spin_unlock(spinlock_t *lock);

/*上锁并关中断*/
void spin_lock_irq(spinlock_t *lock);

/*解锁并开中断*/
void spin_unlock_irq(spinlock_t *lock);

/*上锁并关底半部*/
void spin_lock_bh(spinlock_t *lock);

/*解锁并开底半部*/
void spin_unlock_bh(spinlock_t *lock);
```

自旋锁一般这样被使用:
```c
/*定义一个自旋锁*/
spinlock_t lock;
spin_lock_init(&lock);

spin_lock (&lock); 
	/* 临界区*/
spin_unlock (&lock); 
```


## 读写自旋锁
读写自旋锁(rwlock)只能有1个写操作，允许读的并发。


## 顺序锁
顺序锁(seqlock)是对读写锁的一种优化，读执行单元不必等待写执行单元完成写操作，写执行单元也不需要等待所有读执行单元完成读操作。写执行单元与写执行单元之间仍然是互斥的。如果读执行单元在读操作期间，写执行单元已经发生了写操作，那么读执行单元必须重新读取数据，以便确保得到的数据是完整的。

顺序锁要求被保护的共享资源不含有指针，因为写执行单元可能使得指针失效，但读执行单元如果正要访问该指针，将导致oops。


## 信号量
信号量(semaphore)是一种用于保护临界区的方法，只有得到信号量的进程才能执行临界区代码。当获取不到信号量时，进程会将自身加入一个等待队列中去**睡眠**，当信号量释放后才被唤醒。

信号量的实现依赖于自旋锁和等待队列，为了保证信号量结构存取的原子性，在多CPU中需要自旋锁来互斥。

信号量是进程级的，会导致进程睡眠，睡眠需要进程上下文切换，开销也很大，因此只有当进程占用资源时间较长时，用信号量比较好。在中断或软中断情况下，因为不能被睡眠，所以不能用信号量。

```c
struct semaphore {
	spinlock_t		lock;
	unsigned int		count;
	struct list_head	wait_list;
};

/*初始化信号量，一般为1*/
void sema_init(struct semaphore *sem, int val);

/*获得信号量，它会导致睡眠,因此不能在中断上下文使用*/
void down(struct semaphore *sem);

/*能被信号打断，信号会导致该函数返回非0*/
int down_interruptible(struct semaphore *sem);

/*尝试获得信号量，如果能够立刻获得，它就获得该信号量并返回0,否则返回非0值。它不会导致调用者睡眠，可以在中断上下文使用*/
int down_trylock(struct semaphore *sem);

/*释放信号量*/
void up(struct semaphore *sem);
```

信号量根据count的値，设定可以允许有多少个进程持有这个信号量，一个持有者的信号量叫互斥信号量，多个持有者的信号量叫计数信号量。自旋锁只允许一个持有者。

使用down_interruptible()获取信号量时，要对返回值进行检查，如果非0立即返回-ERESTARTSYS。
```c
if (down_interruptible(&sem)) 
	return -ERESTARTSYS;
```

## 完成量
```c
struct completion {
	unsigned int done;
	wait_queue_head_t wait;
};

/*初始化完成量*/
void init_completion(struct completion *x);
DECLARE_COMPLETION(x)

/*等待完成量被唤醒*/
void wait_for_completion(struct completion *c);

/*唤醒完成量*/
void complete(struct completion *c);
void complete_all(struct completion *c);
```


## 读写信号量
```c
struct rw_semaphore {
	__s32			activity;
	spinlock_t		wait_lock;
	struct list_head	wait_list;
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	struct lockdep_map dep_map;
#endif
};

/*初始化读写信号量*/
void init_rwsem(struct rw_semaphore *sem);

/*读信号量获取*/
void down_read(struct rw_semaphore *sem);
int down_read_trylock(struct rw_semaphore *sem);

/*读信号量释放*/
void up_read(struct rw_semaphore *sem);

/*写信号量获取*/
void down_write(struct rw_semaphore *sem);
int down_write_trylock(struct rw_semaphore *sem);

/*写信号量释放*/
void up_write(struct rw_semaphore *sem);
```


## 互斥锁
```c
struct mutex {
	/* 1: unlocked, 0: locked, negative: locked, possible waiters */
	atomic_t		count;
	spinlock_t		wait_lock;
	struct list_head	wait_list;
#if defined(CONFIG_DEBUG_MUTEXES) || defined(CONFIG_SMP)
	struct task_struct	*owner;
#endif
#ifdef CONFIG_DEBUG_MUTEXES
	const char 		*name;
	void			*magic;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	struct lockdep_map	dep_map;
#endif
};

/*互斥体并初始化*/
mutex_init(&mutex);

/*获取互斥体*/
void __sched mutex_lock(struct mutex *lock);
int _ _sched mutex_lock_interruptible(struct mutex *lock);
int _ _sched mutex_trylock(struct mutex *lock);

/*释放互斥体*/
void __sched mutex_unlock(struct mutex *lock);
```


