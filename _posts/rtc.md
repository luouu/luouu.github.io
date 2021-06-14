<!--
 * @Description: RTC
 * @Author: luo_u
 * @Date: 2020-07-26 08:18:08
 * @LastEditTime: 2020-07-26 16:57:04
--> 

[TOC]

RTC(实时钟)借助电池供电,在系统掉电的情况下时间依然可以正常走动。它通常还具有产生周期性中断以及产生闹钟(alarm)中断的能力。

![](.img/rtc/rtc-arch.png)

- rtc-dev.c   rtc设备用户接口;
- class.c     管理rtc sys class;
- interface.c 间接rtc驱动接口;
- rtc-lib.c   rtc辅助函数,主要用于rtc时间转换、计算;
- rtc-proc    rtc proc fs用户接口;
- rtc-sysfs   rtc sys fs用户接口;
- hctosys.c   用于在系统启动时从rtc读取时间，并设置为系统时间;
- rtc-xxx.c   rtc硬件芯片驱动，如rtc-wm8350.c、rtc-s3c.c等。
- include/linux/rtc.h  rtc接口声明，定义ioctl命令宏。

RTC框架大致可以分为三部分：芯片底层驱动，核心层API以及应用层接口，它们依次向上抽象注册。同时又对下面底层的函数进行回调。

## rtc-xxx.c
不同的芯片对应不同的.c文件，每种芯片的实现方式都不一样，由芯片硬件自身决定。

```c
struct rtc_class_ops {
	int (*open)(struct device *);
	void (*release)(struct device *);
	int (*ioctl)(struct device *, unsigned int, unsigned long);
	int (*read_time)(struct device *, struct rtc_time *);
	int (*set_time)(struct device *, struct rtc_time *);
	int (*read_alarm)(struct device *, struct rtc_wkalrm *);
	int (*set_alarm)(struct device *, struct rtc_wkalrm *);
	int (*proc)(struct device *, struct seq_file *);
	int (*set_mmss)(struct device *, unsigned long secs);
	int (*read_callback)(struct device *, int data);
	int (*alarm_irq_enable)(struct device *, unsigned int enabled);
};

static const struct rtc_class_ops s3c_rtcops = {
	.read_time	= s3c_rtc_gettime,
	.set_time	= s3c_rtc_settime,
	.read_alarm	= s3c_rtc_getalarm,
	.set_alarm	= s3c_rtc_setalarm,
	.alarm_irq_enable = s3c_rtc_setaie,
};

struct rtc_device *rtc;
rtc = rtc_device_register("s3c", &pdev->dev, &s3c_rtcops, THIS_MODULE);
```
rtc芯片要实例化一个 rtc_class_ops 结构体，里面是一些rtc设备的操作函数，然后使用 [rtc_device_register()](#rtc_device_register) 函数注册到rtc设备中。



## rtc-class.c
rtc-class.c 提供了一些rtc核心的API。

```c

struct rtc_device  
{  
    struct device dev;  
    struct module *owner;  
  
    int id;                                              //代表是那个rtc设备  
    char name[RTC_DEVICE_NAME_SIZE];                     //代表rtc设备的名称  
  
    const struct rtc_class_ops *ops;                     //rtc操作函数集，需要驱动实现  
    struct mutex ops_lock;                               //操作函数集的互斥锁  
  
    struct cdev char_dev;                                //代表rtc字符设备，因为rtc就是个字符设备  
    unsigned long flags;                                 //rtc的状态标志，例如RTC_DEV_BUSY  
  
    unsigned long irq_data;                              //rtc中断数据  
    spinlock_t irq_lock;                                 //访问数据是要互斥，需要spin_lock  
    wait_queue_head_t irq_queue;                         //数据查询中用到rtc队列  
    struct fasync_struct *async_queue;                   //异步队列  
  
    struct rtc_task *irq_task;                           //在中断中使用task传输数据  
    spinlock_t irq_task_lock;                            //task传输互斥  
    int irq_freq;                                        //rtc的中断频率  
    int max_user_freq;                                   //rtc的最大中断频率  
  
    struct timerqueue_head timerqueue;                   //定时器队列                    
    struct rtc_timer aie_timer;                          //aie(alaram interrupt enable)定时器  
    struct rtc_timer uie_rtctimer;                       //uie(update interrupt enable)定时器  
    struct hrtimer pie_timer; /* sub second exp, so needs hrtimer */ //pie(periodic interrupt enable)定时器  
    int pie_enabled;                                     //pie使能标志  
    struct work_struct irqwork;                            
    /* Some hardware can't support UIE mode */  
    int uie_unsupported;                                 //uie使能标志  
  
#ifdef CONFIG_RTC_INTF_DEV_UIE_EMUL                      //RTC UIE emulation on dev interface配置项，目前没有开启  
    struct work_struct uie_task;  
    struct timer_list uie_timer;  
    /* Those fields are protected by rtc->irq_lock */  
    unsigned int oldsecs;  
    unsigned int uie_irq_active:1;  
    unsigned int stop_uie_polling:1;  
    unsigned int uie_task_active:1;  
    unsigned int uie_timer_active:1;  
#endif  

————————————————
版权声明：本文为CSDN博主「moxue10」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/W1107101310/java/article/details/79947761
```

### rtc_device_register
```c
struct rtc_device *rtc_device_register(const char *name, struct device *dev,
					const struct rtc_class_ops *ops,
					struct module *owner)
{
	struct rtc_device *rtc;
	struct rtc_wkalrm alrm;
	int id, err;

	if (idr_pre_get(&rtc_idr, GFP_KERNEL) == 0) {
		err = -ENOMEM;
		goto exit;
	}


	mutex_lock(&idr_lock);
	err = idr_get_new(&rtc_idr, NULL, &id);
	mutex_unlock(&idr_lock);

	if (err < 0)
		goto exit;

	id = id & MAX_ID_MASK;

	rtc = kzalloc(sizeof(struct rtc_device), GFP_KERNEL);
	if (rtc == NULL) {
		err = -ENOMEM;
		goto exit_idr;
	}

	rtc->id = id;
	rtc->ops = ops;
	rtc->owner = owner;
	rtc->irq_freq = 1;
	rtc->max_user_freq = 64;
	rtc->dev.parent = dev;
	rtc->dev.class = rtc_class;
	rtc->dev.release = rtc_device_release;

	mutex_init(&rtc->ops_lock);
	spin_lock_init(&rtc->irq_lock);
	spin_lock_init(&rtc->irq_task_lock);
	init_waitqueue_head(&rtc->irq_queue);

	/* Init timerqueue */
	timerqueue_init_head(&rtc->timerqueue);
	INIT_WORK(&rtc->irqwork, rtc_timer_do_work);
	/* Init aie timer */
	rtc_timer_init(&rtc->aie_timer, rtc_aie_update_irq, (void *)rtc);
	/* Init uie timer */
	rtc_timer_init(&rtc->uie_rtctimer, rtc_uie_update_irq, (void *)rtc);
	/* Init pie timer */
	hrtimer_init(&rtc->pie_timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL);
	rtc->pie_timer.function = rtc_pie_update_irq;
	rtc->pie_enabled = 0;

	/* Check to see if there is an ALARM already set in hw */
	err = __rtc_read_alarm(rtc, &alrm);

	if (!err && !rtc_valid_tm(&alrm.time))
		rtc_initialize_alarm(rtc, &alrm);

	strlcpy(rtc->name, name, RTC_DEVICE_NAME_SIZE);
	dev_set_name(&rtc->dev, "rtc%d", id);

	rtc_dev_prepare(rtc);

	err = device_register(&rtc->dev);
	if (err) {
		put_device(&rtc->dev);
		goto exit_kfree;
	}

	rtc_dev_add_device(rtc);
	rtc_sysfs_add_device(rtc);
	rtc_proc_add_device(rtc);

	dev_info(dev, "rtc core: registered %s as %s\n",
			rtc->name, dev_name(&rtc->dev));

	return rtc;

exit_kfree:
	kfree(rtc);

exit_idr:
	mutex_lock(&idr_lock);
	idr_remove(&rtc_idr, id);
	mutex_unlock(&idr_lock);

exit:
	dev_err(dev, "rtc core: unable to register %s, err = %d\n",
			name, err);
	return ERR_PTR(err);
```
rtc_device_register()函数主要工作是创建一个rtc设备和实现proc、sysfs文件系统操作。



## rtc-dev.c
rtc-dev.c 是硬件抽象层，为应用层提供了文件操作接口，如read, ioctl等。同时也为rtc-class.c 提供了API，rtc字符设备的创建的实现也在这里。

```c
#define RTC_AIE_ON	_IO('p', 0x01)	/* Alarm int. enable on		*/
#define RTC_AIE_OFF	_IO('p', 0x02)	/* ... off			*/
#define RTC_UIE_ON	_IO('p', 0x03)	/* Update int. enable on	*/
#define RTC_UIE_OFF	_IO('p', 0x04)	/* ... off			*/
#define RTC_PIE_ON	_IO('p', 0x05)	/* Periodic int. enable on	*/
#define RTC_PIE_OFF	_IO('p', 0x06)	/* ... off			*/
#define RTC_WIE_ON	_IO('p', 0x0f)  /* Watchdog int. enable on	*/
#define RTC_WIE_OFF	_IO('p', 0x10)  /* ... off			*/

#define RTC_ALM_SET	_IOW('p', 0x07, struct rtc_time) /* Set alarm time  */
#define RTC_ALM_READ	_IOR('p', 0x08, struct rtc_time) /* Read alarm time */
#define RTC_RD_TIME	_IOR('p', 0x09, struct rtc_time) /* Read RTC time   */
#define RTC_SET_TIME	_IOW('p', 0x0a, struct rtc_time) /* Set RTC time    */
#define RTC_IRQP_READ	_IOR('p', 0x0b, unsigned long)	 /* Read IRQ rate   */
#define RTC_IRQP_SET	_IOW('p', 0x0c, unsigned long)	 /* Set IRQ rate    */
#define RTC_EPOCH_READ	_IOR('p', 0x0d, unsigned long)	 /* Read epoch      */
#define RTC_EPOCH_SET	_IOW('p', 0x0e, unsigned long)	 /* Set epoch       */

#define RTC_WKALM_SET	_IOW('p', 0x0f, struct rtc_wkalrm)/* Set wakeup alarm*/
#define RTC_WKALM_RD	_IOR('p', 0x10, struct rtc_wkalrm)/* Get wakeup alarm*/

#define RTC_PLL_GET	_IOR('p', 0x11, struct rtc_pll_info)  /* Get PLL correction */
#define RTC_PLL_SET	_IOW('p', 0x12, struct rtc_pll_info)  /* Set PLL correction */
```

```c
static long rtc_dev_ioctl(struct file *file,
		unsigned int cmd, unsigned long arg)
{
	int err = 0;
	struct rtc_device *rtc = file->private_data;
	const struct rtc_class_ops *ops = rtc->ops;
	struct rtc_time tm;
	struct rtc_wkalrm alarm;
	void __user *uarg = (void __user *) arg;

	err = mutex_lock_interruptible(&rtc->ops_lock);
	if (err)
		return err;

	/* check that the calling task has appropriate permissions
	 * for certain ioctls. doing this check here is useful
	 * to avoid duplicate code in each driver.
	 */
	switch (cmd) {
	case RTC_EPOCH_SET:
	case RTC_SET_TIME:
		if (!capable(CAP_SYS_TIME))
			err = -EACCES;
		break;

	case RTC_IRQP_SET:
		if (arg > rtc->max_user_freq && !capable(CAP_SYS_RESOURCE))
			err = -EACCES;
		break;

	case RTC_PIE_ON:
		if (rtc->irq_freq > rtc->max_user_freq &&
				!capable(CAP_SYS_RESOURCE))
			err = -EACCES;
		break;
	}

	if (err)
		goto done;

	/*
	 * Drivers *SHOULD NOT* provide ioctl implementations
	 * for these requests.  Instead, provide methods to
	 * support the following code, so that the RTC's main
	 * features are accessible without using ioctls.
	 *
	 * RTC and alarm times will be in UTC, by preference,
	 * but dual-booting with MS-Windows implies RTCs must
	 * use the local wall clock time.
	 */

	switch (cmd) {
	case RTC_ALM_READ:
		mutex_unlock(&rtc->ops_lock);

		err = rtc_read_alarm(rtc, &alarm);
		if (err < 0)
			return err;

		if (copy_to_user(uarg, &alarm.time, sizeof(tm)))
			err = -EFAULT;
		return err;

	case RTC_ALM_SET:
		mutex_unlock(&rtc->ops_lock);

		if (copy_from_user(&alarm.time, uarg, sizeof(tm)))
			return -EFAULT;

		alarm.enabled = 0;
		alarm.pending = 0;
		alarm.time.tm_wday = -1;
		alarm.time.tm_yday = -1;
		alarm.time.tm_isdst = -1;

		/* RTC_ALM_SET alarms may be up to 24 hours in the future.
		 * Rather than expecting every RTC to implement "don't care"
		 * for day/month/year fields, just force the alarm to have
		 * the right values for those fields.
		 *
		 * RTC_WKALM_SET should be used instead.  Not only does it
		 * eliminate the need for a separate RTC_AIE_ON call, it
		 * doesn't have the "alarm 23:59:59 in the future" race.
		 *
		 * NOTE:  some legacy code may have used invalid fields as
		 * wildcards, exposing hardware "periodic alarm" capabilities.
		 * Not supported here.
		 */
		{
			unsigned long now, then;

			err = rtc_read_time(rtc, &tm);
			if (err < 0)
				return err;
			rtc_tm_to_time(&tm, &now);

			alarm.time.tm_mday = tm.tm_mday;
			alarm.time.tm_mon = tm.tm_mon;
			alarm.time.tm_year = tm.tm_year;
			err  = rtc_valid_tm(&alarm.time);
			if (err < 0)
				return err;
			rtc_tm_to_time(&alarm.time, &then);

			/* alarm may need to wrap into tomorrow */
			if (then < now) {
				rtc_time_to_tm(now + 24 * 60 * 60, &tm);
				alarm.time.tm_mday = tm.tm_mday;
				alarm.time.tm_mon = tm.tm_mon;
				alarm.time.tm_year = tm.tm_year;
			}
		}

		return rtc_set_alarm(rtc, &alarm);

	case RTC_RD_TIME:
		mutex_unlock(&rtc->ops_lock);

		err = rtc_read_time(rtc, &tm);
		if (err < 0)
			return err;

		if (copy_to_user(uarg, &tm, sizeof(tm)))
			err = -EFAULT;
		return err;

	case RTC_SET_TIME:
		mutex_unlock(&rtc->ops_lock);

		if (copy_from_user(&tm, uarg, sizeof(tm)))
			return -EFAULT;

		return rtc_set_time(rtc, &tm);

	case RTC_PIE_ON:
		err = rtc_irq_set_state(rtc, NULL, 1);
		break;

	case RTC_PIE_OFF:
		err = rtc_irq_set_state(rtc, NULL, 0);
		break;

	case RTC_AIE_ON:
		mutex_unlock(&rtc->ops_lock);
		return rtc_alarm_irq_enable(rtc, 1);

	case RTC_AIE_OFF:
		mutex_unlock(&rtc->ops_lock);
		return rtc_alarm_irq_enable(rtc, 0);

	case RTC_UIE_ON:
		mutex_unlock(&rtc->ops_lock);
		return rtc_update_irq_enable(rtc, 1);

	case RTC_UIE_OFF:
		mutex_unlock(&rtc->ops_lock);
		return rtc_update_irq_enable(rtc, 0);

	case RTC_IRQP_SET:
		err = rtc_irq_set_freq(rtc, NULL, arg);
		break;

	case RTC_IRQP_READ:
		err = put_user(rtc->irq_freq, (unsigned long __user *)uarg);
		break;

	case RTC_WKALM_SET:
		mutex_unlock(&rtc->ops_lock);
		if (copy_from_user(&alarm, uarg, sizeof(alarm)))
			return -EFAULT;

		return rtc_set_alarm(rtc, &alarm);

	case RTC_WKALM_RD:
		mutex_unlock(&rtc->ops_lock);
		err = rtc_read_alarm(rtc, &alarm);
		if (err < 0)
			return err;

		if (copy_to_user(uarg, &alarm, sizeof(alarm)))
			err = -EFAULT;
		return err;

	default:
		/* Finally try the driver's ioctl interface */
		if (ops->ioctl) {
			err = ops->ioctl(rtc->dev.parent, cmd, arg);
			if (err == -ENOIOCTLCMD)
				err = -ENOTTY;
		} else
			err = -ENOTTY;
		break;
	}

done:
	mutex_unlock(&rtc->ops_lock);
	return err;
```


