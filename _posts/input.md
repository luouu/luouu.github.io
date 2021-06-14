## 输入子系统框架

Linux输入子系统由Input driver（驱动层）、Input core（输入子系统核心）、Event handler（事件处理层）3部分组成。

![](.img/input/arch2.jpg)

![Linux的input输入子系统:总体框架 ](.img/input/arch.jpg)

## 设备驱动层

设备驱动层主要实现对硬件设备的读写访问，中断设置，并把硬件产生的事件通过核心层定义的的API上报给事件处理层。

### input_dev
```c
struct input_dev {
	const char *name;//设备名
	const char *phys;
	const char *uniq;
	struct input_id id;  //与 handler 匹配

	/* 输入设备支持事件的位图*/
	unsigned long propbit[BITS_TO_LONGS(INPUT_PROP_CNT)];
	unsigned long evbit[BITS_TO_LONGS(EV_CNT)];   // 所有事件
    unsigned long keybit[BITS_TO_LONGS(KEY_CNT)]; // 按键事件
    unsigned long relbit[BITS_TO_LONGS(REL_CNT)]; // 相对位移事件
	unsigned long absbit[BITS_TO_LONGS(ABS_CNT)]; //记录支持的绝对坐标的位图
	unsigned long mscbit[BITS_TO_LONGS(MSC_CNT)];
	unsigned long ledbit[BITS_TO_LONGS(LED_CNT)];
	unsigned long sndbit[BITS_TO_LONGS(SND_CNT)];
	unsigned long ffbit[BITS_TO_LONGS(FF_CNT)];
	unsigned long swbit[BITS_TO_LONGS(SW_CNT)];

	unsigned int hint_events_per_packet;

	unsigned int keycodemax;   //支持的按键值的个数
	unsigned int keycodesize;  //每个键值的字节数
	void *keycode;  //存储按键值的数组首地址

	int (*setkeycode)(struct input_dev *dev, int scancode, int keycode);
	int (*getkeycode)(struct input_dev *dev, int scancode, int *keycode);

	struct ff_device *ff;

	unsigned int repeat_key;   //最近一次按键值，用于连击
	struct timer_list timer;   //自动连击计时器

	int abs[ABS_MAX + 1];
	int rep[REP_MAX + 1];

	struct input_mt_slot *mt;
	int mtsize;
	int slot;
	int trkid;

	struct input_absinfo *absinfo;

	unsigned long key[BITS_TO_LONGS(KEY_CNT)];  //反映当前按键状态的位图
	unsigned long led[BITS_TO_LONGS(LED_CNT)];  //反映当前led状态的位图
	unsigned long snd[BITS_TO_LONGS(SND_CNT)];  //反映当前beep状态的位图
	unsigned long sw[BITS_TO_LONGS(SW_CNT)];

	int absmax[ABS_MAX + 1];
	int absmin[ABS_MAX + 1];
	int absfuzz[ABS_MAX + 1];
	int absflat[ABS_MAX + 1];

	int (*open)(struct input_dev *dev);   //打开函数
	void (*close)(struct input_dev *dev); //关闭函数
	int (*flush)(struct input_dev *dev, struct file *file);  //断开连接时刷新数据
	int (*event)(struct input_dev *dev, unsigned int type, unsigned int code, int value); 

	struct input_handle __rcu *grab;　　//当前占用该设备的 input_handle

	spinlock_t event_lock;
	struct mutex mutex;

	unsigned int users;
	int going_away;
	bool sync;   //最后一次同步后没有新的事件置1
	struct device dev;

	struct list_head h_list;  
	struct list_head node; 
};

struct input_id {
	__u16 bustype;  /*总线类型*/
	__u16 vendor;   /*生产商编号*/
	__u16 product;  /*产品编号*/
	__u16 version;  /* 版本号 */
};
```
input_dev 结构体代表一个具体的硬件设备。node是 input_dev 链表，包含所有注册了的 input_dev 设备。h_list 是handle 链表，包含该设备相匹配的 input_handler 设备处理方法。input_dev 通过设置位图来标记设备具有什么功能。



### 操作流程

1. 实例化input_dev 对象，创建一个硬件设备
```c
struct input_dev *input_allocate_device(void);

void input_free_device(struct input_dev *dev);
```

2. 设置支持的事件类型
```c
set_bit(EV_KEY, input_dev->evbit)

set_bit(KEY_ENTER, buttons_dev->keybit)
```

3.　相关硬件初始化，中断初始化，定义中断处理程序

4. 注册input_dev
```c
int input_register_device(struct input_dev *dev);
void input_unregister_device(struct input_dev *dev);
```

5. 上报事件，一般在中断函数内。
```c
void input_event(struct input_dev *dev,　unsigned int type, unsigned int code, int value)

static inline void input_report_key(struct input_dev *dev, unsigned int code, int value)
{   
  input_event(dev, EV_KEY, code, !!value);
}
  
static inline void input_report_rel(struct input_dev *dev, unsigned int code, int value)
{     
  input_event(dev, EV_REL, code, value);
}
  
static inline void input_report_abs(struct input_dev *dev, unsigned int code, int value)
{   
  input_event(dev, EV_ABS, code, value);
}
  
static inline void input_report_ff_status(struct input_dev *dev, unsigned int code, int value)
{
  input_event(dev, EV_FF_STATUS, code, value);
}   

static inline void input_report_switch(struct input_dev *dev, unsigned int code, int value)
{   
  input_event(dev, EV_SW, code, !!value);
}

static inline void input_sync(struct input_dev *dev)
{
  input_event(dev, EV_SYN, SYN_REPORT, 0);
}
```


### 例程
```c
#include <linux/input.h>
#include <linux/module.h>
#include <linux/init.h>

#include <asm/irq.h>
#include <asm/io.h>

static struct input_dev *button_dev;

static irqreturn_t button_interrupt(int irq, void *dummy)
{
	input_report_key(button_dev, BTN_0, inb(BUTTON_PORT) & 1);
	input_sync(button_dev);
	return IRQ_HANDLED;
}

static int __init button_init(void)
{
	int error;

	if (request_irq(BUTTON_IRQ, button_interrupt, 0, "button", NULL)) {
            printk(KERN_ERR "button.c: Can't allocate irq %d\n", button_irq);
            return -EBUSY;
        }

	button_dev = input_allocate_device();
	if (!button_dev) {
		printk(KERN_ERR "button.c: Not enough memory\n");
		error = -ENOMEM;
		goto err_free_irq;
	}

	button_dev->evbit[0] = BIT_MASK(EV_KEY);
	button_dev->keybit[BIT_WORD(BTN_0)] = BIT_MASK(BTN_0);

	error = input_register_device(button_dev);
	if (error) {
		printk(KERN_ERR "button.c: Failed to register device\n");
		goto err_free_dev;
	}

	return 0;
}

static void __exit button_exit(void)
{
	input_unregister_device(button_dev);
	free_irq(BUTTON_IRQ, button_interrupt);
}

module_init(button_init);
module_exit(button_exit);
```



## 核心层

核心层为事件处理层和设备驱动层提供接口API，通知事件处理层对事件进行处理。具体实现代码在drivers/input/input.c。input核心层维护一个input_device链表和input_handler链表，还有一个input_handler数组。

### input_init()

input_init()完成设备的初始化。创建proc接口，`/proc/bus/input/devices`文件显示已经注册的输入设备；`/proc/bus/input/handlers`文件显示已注册事件处理函数。在/sys/class下创建input类。创建字符设备，主设备号是13，次设备号分布如下：
- joystick游戏杆：0~16
- mouse鼠标： 32~62
- mice鼠标： 63
- 事件设备： 64~95

```c
static int __init input_init(void)
{
	int err;

	err = class_register(&input_class);
	if (err) {
		pr_err("unable to register input_dev class\n");
		return err;
	}

	err = input_proc_init();
	if (err)
		goto fail1;

	err = register_chrdev_region(MKDEV(INPUT_MAJOR, 0),
				     INPUT_MAX_CHAR_DEVICES, "input");
	if (err) {
		pr_err("unable to register char major %d", INPUT_MAJOR);
		goto fail2;
	}
	...
}
```

### input_register_device()

input_register_device()注册输入设备。把输入设备挂到输入设备链表 input_dev_list 中，遍历 input_handler_list 链表，关联对应的事件处理器。
```c
int input_register_device(struct input_dev *dev)
{
	struct input_devres *devres = NULL;
	struct input_handler *handler;

	/* Every input device generates EV_SYN/SYN_REPORT events. */
	__set_bit(EV_SYN, dev->evbit);

	/* KEY_RESERVED is not supposed to be transmitted to userspace. */
	__clear_bit(KEY_RESERVED, dev->keybit);

	/* Make sure that bitmasks not mentioned in dev->evbit are clean. */
	input_cleanse_bitmasks(dev);

	if (!dev->rep[REP_DELAY] && !dev->rep[REP_PERIOD])
		input_enable_softrepeat(dev, 250, 33);

	if (!dev->getkeycode)
		dev->getkeycode = input_default_getkeycode;

	if (!dev->setkeycode)
		dev->setkeycode = input_default_setkeycode;
	...

	list_add_tail(&dev->node, &input_dev_list);

	list_for_each_entry(handler, &input_handler_list, node)
		input_attach_handler(dev, handler);
}
```

### input_register_handler()

input_register_handler()注册事件处理器。把 handler 存放到 input_table[]中，并挂到链表 input_handler_list 中，遍历 input_dev_list 链表，关联对应的输入设备。
```c 
int input_register_handler(struct input_handler *handler)
{
	struct input_dev *dev;
	...

	INIT_LIST_HEAD(&handler->h_list);

	list_add_tail(&handler->node, &input_handler_list);

	list_for_each_entry(dev, &input_dev_list, node)
		input_attach_handler(dev, handler);
	...
}
```

### input_attach_handler()

input_attach_handler()用来匹配 input_dev 和 input_handler ，如果匹配成功，则调用 handler->connnect()将 input_dev 和 input_handler 连接。
```c
static int input_attach_handler(struct input_dev *dev, struct input_handler *handler)
{
	const struct input_device_id *id;
	int error;

	id = input_match_device(handler, dev);
	if (!id)
		return -ENODEV;

	error = handler->connect(handler, dev, id);
	if (error && error != -ENODEV)
		pr_err("failed to attach handler %s to device %s, error: %d\n",
		       handler->name, kobject_name(&dev->dev.kobj), error);

	return error;
}
```

### input_match_device()

input_match_device()匹配 input_dev 和 input_handler 的id成员。如果 id->driver_info 设置了说明它匹配所有的id，evdev就是这个样的。
```c
static const struct input_device_id *input_match_device(struct input_handler *handler,
							struct input_dev *dev)
{
	const struct input_device_id *id;

	for (id = handler->id_table; id->flags || id->driver_info; id++) {
		if (input_match_device_id(dev, id) &&
		    (!handler->match || handler->match(handler, dev))) {
			return id;
		}
	}

	return NULL;
}

struct input_device_id {

	kernel_ulong_t flags;

	__u16 bustype;
	__u16 vendor;
	__u16 product;
	__u16 version;

	kernel_ulong_t evbit[INPUT_DEVICE_ID_EV_MAX / BITS_PER_LONG + 1];
	kernel_ulong_t keybit[INPUT_DEVICE_ID_KEY_MAX / BITS_PER_LONG + 1];
	kernel_ulong_t relbit[INPUT_DEVICE_ID_REL_MAX / BITS_PER_LONG + 1];
	kernel_ulong_t absbit[INPUT_DEVICE_ID_ABS_MAX / BITS_PER_LONG + 1];
	kernel_ulong_t mscbit[INPUT_DEVICE_ID_MSC_MAX / BITS_PER_LONG + 1];
	kernel_ulong_t ledbit[INPUT_DEVICE_ID_LED_MAX / BITS_PER_LONG + 1];
	kernel_ulong_t sndbit[INPUT_DEVICE_ID_SND_MAX / BITS_PER_LONG + 1];
	kernel_ulong_t ffbit[INPUT_DEVICE_ID_FF_MAX / BITS_PER_LONG + 1];
	kernel_ulong_t swbit[INPUT_DEVICE_ID_SW_MAX / BITS_PER_LONG + 1];
	kernel_ulong_t propbit[INPUT_DEVICE_ID_PROP_MAX / BITS_PER_LONG + 1];

	kernel_ulong_t driver_info;
};
```

### input_register_handle()

将自己添加到 input_dev.h_list链表和 input_handler.h_list 链表中，如果定义start函数，则调用它。
```c
int input_register_handle(struct input_handle *handle)
{
	struct input_handler *handler = handle->handler;
	struct input_dev *dev = handle->dev;

	if (handler->filter)
		list_add_rcu(&handle->d_node, &dev->h_list);
	else
		list_add_tail_rcu(&handle->d_node, &dev->h_list);

	list_add_tail_rcu(&handle->h_node, &handler->h_list);

	if (handler->start)
		handler->start(handle);
}
```



## 事件处理层

事件处理层提供用户编程的接口（设备节点），并处理驱动层上报的数据处理。不同类型的输入设备都有相应的实现文件，如evdev.c、mousedev.c、joydev.c。


### input_handler
input_handler结构体代表一种事件处理器，通过设置input_device_id中的位图来标记匹配的设备至少要满足什么功能。

```c
struct input_handler {                     
	void *private;

	/*处理设备驱动报告的事件*/
	void (*event)(struct input_handle *handle, unsigned int type, unsigned int code, int value);
	bool (*filter)(struct input_handle *handle, unsigned int type, unsigned int code, int value);
	bool (*match)(struct input_handler *handler, struct input_dev *dev);
	int (*connect)(struct input_handler *handler, struct input_dev *dev, 
		const struct input_device_id *id);
	void (*disconnect)(struct input_handle *handle);

	void (*start)(struct input_handle *handle);

	const struct file_operations *fops;
	int minor;
	const char *name;

	const struct input_device_id *id_table; //存放该handler所支持的设备id的表

	struct list_head  h_list;
	struct list_head  node;
};
```


### input_handle
input_handle 结构体代表一个成功配对的input_dev和input_handler。input_hande 没有一个全局的链表，它注册的时候将自己分别挂在了 input_device 和 input_handler 的h_list上了；同时，input_handle结构体成员dev关联到input_dev结构，handler关联到input_handler结构。

当注册一个input_device的时候，会遍历input_hanlder链表，查看是否匹配，如果匹配，就会创建一个input_hanle连接器，然后分别存放到对应的input_devide和input_handler中的input_handle链表中，注册input_handler的时候也是同理。
```c
struct input_handle {                                                                                             
	void *private;

	int open;  //设备打开次数
	const char *name;

	struct input_dev *dev;
	struct input_handler *handler;

	struct list_head  d_node;
	struct list_head  h_node;
};
```

### connect()

每个 input_handler 都会实现相应的 connect 方法。里面会调用 input_register_handle()注册一个input_handle。
```c
static int evdev_connect(struct input_handler *handler, struct input_dev *dev,
			 const struct input_device_id *id)
{
	struct evdev *evdev;
	int minor;
	int dev_no;
	int error;

	minor = input_get_new_minor(EVDEV_MINOR_BASE, EVDEV_MINORS, true);
	if (minor < 0) {
		error = minor;
		pr_err("failed to reserve new minor: %d\n", error);
		return error;
	}

	evdev = kzalloc(sizeof(struct evdev), GFP_KERNEL);
	if (!evdev) {
		error = -ENOMEM;
		goto err_free_minor;
	}
	...

	evdev->handle.dev = input_get_device(dev);
	evdev->handle.name = dev_name(&evdev->dev);
	evdev->handle.handler = handler;
	evdev->handle.private = evdev;

	evdev->dev.devt = MKDEV(INPUT_MAJOR, minor);
	evdev->dev.class = &input_class;
	evdev->dev.parent = &dev->dev;
	evdev->dev.release = evdev_free;
	device_initialize(&evdev->dev);

	error = input_register_handle(&evdev->handle);
	if (error)
		goto err_free_evdev;

	cdev_init(&evdev->cdev, &evdev_fops);

	error = cdev_device_add(&evdev->cdev, &evdev->dev);
	if (error)
		goto err_cleanup_evdev;
	...
}
```

### read()

当应用层调用read函数时，将调用到 evdev_read()函数。如果进入了休眠状态，将由 evdev_event()函数唤醒，而这些都是驱动层上报事件时调用input_event()函数引起的。
```c
static ssize_t evdev_read(struct file *file, char __user *buffer,
			  size_t count, loff_t *ppos)
{
	struct evdev_client *client = file->private_data;
	struct evdev *evdev = client->evdev;
	struct input_event event;
	size_t read = 0;
	int error;

	if (count != 0 && count < input_event_size())
		return -EINVAL;

	for (;;) {
		if (!evdev->exist || client->revoked)
			return -ENODEV;

		if (client->packet_head == client->tail &&
		    (file->f_flags & O_NONBLOCK))
			return -EAGAIN;

		/*
		 * count == 0 is special - no IO is done but we check
		 * for error conditions (see above).
		 */
		if (count == 0)
			break;

		while (read + input_event_size() <= count &&
		       evdev_fetch_next_event(client, &event)) {

			if (input_event_to_user(buffer + read, &event))
				return -EFAULT;

			read += input_event_size();
		}

		if (read)
			break;

		if (!(file->f_flags & O_NONBLOCK)) {
			error = wait_event_interruptible(evdev->wait,
					client->packet_head != client->tail ||
					!evdev->exist || client->revoked);
			if (error)
				return error;
		}
	}

	return read;
}
```



## 应用层获取

input产生的事件用struct input_event表示，应用层只要调read()函数读/dev/input/下相应设备节点，从而获取到input_event信息。

```c
struct input_event {
	struct timeval time; //事件发生的时间
	unsigned short type; //事件的类型
	unsigned short code; //事件的代码
	unsigned int value;  //事件的值
};
```




