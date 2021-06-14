<!--
 * @Description: i2c
 * @Author: luo_u
 * @Date: 2020-07-30 14:57:37
 * @LastEditTime: 2020-08-19 19:38:55
--> 

[TOC]

## i2c 框架

![i2c体系结构](.img/i2c/i2c-arch.webp)

![i2c分层结构](.img/i2c/i2c-arch.png)

```
├── algos　　　　　　　　　　　实现了一些i2c总线适配器的算法
│   ├── i2c-algo-bit.c
│   ├── i2c-algo-pca.c
│   ├── i2c-algo-pcf.c
│   ├── i2c-algo-pcf.h
├── busses　　　　　　　　　　不同处理器的i2c控制器驱动
│   ├── i2c-at91.c
│   ├── i2c-exynos5.c
│   ├── i2c-fsi.c
│   ├── i2c-gpio.c
│   ├── i2c-s3c2410.c
│   ├── i2c-stm32.c
├── chips                  设备器驱动　
├── i2c-boardinfo.c
├── i2c-core-acpi.c
├── i2c-core-base.c
├── i2c-core.h
├── i2c-core-of.c
├── i2c-core-slave.c
├── i2c-core-smbus.c
├── i2c-dev.c
├── i2c-mux.c
├── i2c-slave-eeprom.c
├── i2c-smbus.c
├── i2c-stub.c
└── muxes
    ├── i2c-arb-gpio-challenge.c
    ├── i2c-demux-pinctrl.c
    ├── i2c-mux-gpio.c
    ├── i2c-mux-gpmux.c
    ├── i2c-mux-ltc4306.c
    ├── i2c-mux-mlxcpld.c
    ├── i2c-mux-pca9541.c
    ├── i2c-mux-pca954x.c
    ├── i2c-mux-pinctrl.c
    ├── i2c-mux-reg.c
```

## i2c 核心

I2C核心维护了i2c_bus结构体，提供了I2C总线驱动和设备驱动的注册、注销方法，维护了I2C总线的驱动、设备链表，实现了设备、驱动的匹配探测。代码实现在i2c-core.c中。

### 注册i2c总线
i2c总线用于管理I2C设备和I2C驱动，维护一个设备链表和驱动链表，定义了设备和驱动的匹配规则和匹配成功后的行为。

```c
struct bus_type i2c_bus_type = {
	.name		= "i2c",
	.match		= i2c_device_match,
	.probe		= i2c_device_probe,
	.remove		= i2c_device_remove,
	.shutdown	= i2c_device_shutdown,
};

ret = bus_register(&i2c_bus_type);
```
i2c_device_match()中，会调用 i2c_match_id()函数匹配板文件中定义的ID和i2c_driver 所支持的ID表。



### i2c_add_adapter()

```c
int i2c_add_adapter(struct i2c_adapter *adapter)
{
	struct device *dev = &adapter->dev;
	int id;

	mutex_lock(&core_lock);
	id = idr_alloc(&i2c_adapter_idr, adapter,
		       __i2c_first_dynamic_bus_num, 0, GFP_KERNEL);
	mutex_unlock(&core_lock);
	if (WARN(id < 0, "couldn't get idr"))
		return id;

	adapter->nr = id;

	return i2c_register_adapter(adapter);
}

struct bus_type i2c_bus_type = {
	.name		= "i2c",
	.match		= i2c_device_match,
	.probe		= i2c_device_probe,
	.remove		= i2c_device_remove,
	.shutdown	= i2c_device_shutdown,
};

struct device_type i2c_adapter_type = {
	.groups		= i2c_adapter_groups,
	.release	= i2c_adapter_dev_release,
};

static int __process_new_adapter(struct device_driver *d, void *data)
{
	return i2c_do_add_adapter(to_i2c_driver(d), data);
}

static int i2c_register_adapter(struct i2c_adapter *adap)
{
	rt_mutex_init(&adap->bus_lock);
	rt_mutex_init(&adap->mux_lock);
	mutex_init(&adap->userspace_clients_lock);
	INIT_LIST_HEAD(&adap->userspace_clients);

	/* Set default timeout to 1 second if not already set */
	if (adap->timeout == 0)
		adap->timeout = HZ;

	/* register soft irqs for Host Notify */
	res = i2c_setup_host_notify_irq_domain(adap);

	dev_set_name(&adap->dev, "i2c-%d", adap->nr);
	adap->dev.bus = &i2c_bus_type;
	adap->dev.type = &i2c_adapter_type;
	res = device_register(&adap->dev);

	i2c_init_recovery(adap);

	of_i2c_register_devices(adap);
	i2c_acpi_register_devices(adap);
	i2c_acpi_install_space_handler(adap);

	if (adap->nr < __i2c_first_dynamic_bus_num)
		i2c_scan_static_board_info(adap);

	mutex_lock(&core_lock);
	bus_for_each_drv(&i2c_bus_type, NULL, adap, __process_new_adapter);
	mutex_unlock(&core_lock);

	return 0;
}
```



### i2c_transfer()

i2c_transfer()函数用于进行i2c适配器和i2c设备之间的消息交互，i2c_master_send()函数和i2c_master_recv()函数内部会调用i2c_transfer()函数分别完成一条写消息和一条读消息。

```c
int i2c_transfer(struct i2c_adapter *adap, struct i2c_msg *msgs, int num)
{
	if (!adap->algo->master_xfer) {
		dev_dbg(&adap->dev, "I2C level transfers not supported\n");
		return -EOPNOTSUPP;
	}

    adap->algo->master_xfer(adap, msgs, num); //调用适配器的算法
}
```
i2c_transfer()函数通过i2c_adapter对应的i2c_algorithm中的master_xfer()函数来传输I2C数据。



### i2c_new_device()
```c
struct i2c_client *
i2c_new_device(struct i2c_adapter *adap, struct i2c_board_info const *info)
{
	struct i2c_client	*client;
	int			status;

	client = kzalloc(sizeof *client, GFP_KERNEL);
	if (!client)
		return NULL;

	client->adapter = adap;

	client->dev.platform_data = info->platform_data;
	client->flags = info->flags;
	client->addr = info->addr;

	client->init_irq = info->irq;
	if (!client->init_irq)
		client->init_irq = i2c_dev_irq_from_resources(info->resources,
							 info->num_resources);
	client->irq = client->init_irq;

	strlcpy(client->name, info->type, sizeof(client->name));

	status = i2c_check_addr_validity(client->addr, client->flags);

	/* Check for address business */
	status = i2c_check_addr_busy(adap, i2c_encode_flags_to_addr(client));

	client->dev.parent = &client->adapter->dev;
	client->dev.bus = &i2c_bus_type;
	client->dev.type = &i2c_client_type;
	client->dev.of_node = of_node_get(info->of_node);
	client->dev.fwnode = info->fwnode;

	i2c_dev_set_name(adap, client, info);

	if (info->properties) {
		status = device_add_properties(&client->dev, info->properties);
		if (status) {
			dev_err(&adap->dev,
				"Failed to add properties to client %s: %d\n",
				client->name, status);
			goto out_err_put_of_node;
		}
	}

	status = device_register(&client->dev);

	return client;
}

```
将i2c设备加入总线的设备链表中，遍历总线的驱动链表上的所有驱动，调用总线的匹配函数，判断I2C驱动的id_table中每一的name和I2C设备的name是否匹配，如果匹配，就调用驱动的probe()函数。



### i2c_register_board_info()

```c
int __init i2c_register_board_info(int busnum, 
	struct i2c_board_info const *info, unsigned len)
{
	int status;

	down_write(&__i2c_board_lock);

	/* dynamic bus numbers will be assigned after the last static one */
	if (busnum >= __i2c_first_dynamic_bus_num)
		__i2c_first_dynamic_bus_num = busnum + 1;

	for (status = 0; len; len--, info++) {
		struct i2c_devinfo	*devinfo;

		devinfo = kzalloc(sizeof(*devinfo), GFP_KERNEL);
		if (!devinfo) {
			pr_debug("i2c-core: can't register boardinfo!\n");
			status = -ENOMEM;
			break;
		}

		devinfo->busnum = busnum;
		devinfo->board_info = *info;
		list_add_tail(&devinfo->list, &__i2c_board_list);
	}

	up_write(&__i2c_board_lock);

	return status;
}
```



### i2c_add_driver()
```c
#define i2c_add_driver(driver) \
	i2c_register_driver(THIS_MODULE, driver)

static int __process_new_driver(struct device *dev, void *data)
{
	if (dev->type != &i2c_adapter_type)
		return 0;
	return i2c_do_add_adapter(data, to_i2c_adapter(dev));
}

int i2c_register_driver(struct module *owner, struct i2c_driver *driver)
{
	driver->driver.owner = owner;
	driver->driver.bus = &i2c_bus_type;
	INIT_LIST_HEAD(&driver->clients);

	res = driver_register(&driver->driver);

	i2c_for_each_dev(driver, __process_new_driver);

	return 0;
}
```
指定总线类型为 i2c_bus_type，绑定总线，向内核注册i2c驱动时，将i2c驱动添加到总线的链表中，遍历总线上所有设备，匹配i2c_client->name, i2c_driver->i2c_device_id->name字符串，如果匹配，就调用驱动程序的probe函数。

遍历总线的设备，调用__process_new_driver()函数。遍历总线上的所有设备，拿到设备对应的适配器，去给总线发送驱动指定好的地址，如果地址存在的话，就调用驱动的detect()函数。



## i2c 总线驱动

I2C总线驱动维护了I2C适配器数据结构（i2c_adapter）和适配器的通信方法数据结构（i2c_algorithm）。所以I2C总线驱动可控制I2C适配器产生start、stop、ACK等。此部分代码由具体的芯片厂商提供。


### i2c_adapter
i2c_adapter 对应与物理上的一个适配器，适配器就是cpu内部的i2c控制器，用来产生总线操作信号。一个适配器上可以连接多个i2c设备，所以i2c_adapter中包含依附于它的i2c_client的链表。

```c
struct i2c_adapter {
	struct module *owner;
	unsigned int class;		  /* classes to allow probing for */
	const struct i2c_algorithm *algo; /* the algorithm to access the bus */
	void *algo_data;

	/* data fields that are valid for all devices	*/
	const struct i2c_lock_operations *lock_ops;
	struct rt_mutex bus_lock;
	struct rt_mutex mux_lock;

	int timeout;			/* in jiffies */
	int retries;
	struct device dev;		/* the adapter device */

	int nr;
	char name[48];
	struct completion dev_released;

	struct mutex userspace_clients_lock;
	struct list_head userspace_clients;

	struct i2c_bus_recovery_info *bus_recovery_info;
	const struct i2c_adapter_quirks *quirks;

	struct irq_domain *host_notify_domain;
};
```



### 注册总线驱动

```c
static int s3c24xx_i2c_probe(struct platform_device *pdev)
{
	struct s3c24xx_i2c *i2c;
	...

	strlcpy(i2c->adap.name, "s3c2410-i2c", sizeof(i2c->adap.name));
	i2c->adap.owner = THIS_MODULE;
	i2c->adap.algo = &s3c24xx_i2c_algorithm;
	i2c->adap.retries = 2;
	i2c->adap.class = I2C_CLASS_DEPRECATED;
	i2c->tx_setup = 50;

	init_waitqueue_head(&i2c->wait);

	i2c->dev = &pdev->dev;
	i2c->clk = devm_clk_get(&pdev->dev, "i2c");

	res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
	i2c->regs = devm_ioremap_resource(&pdev->dev, res);

	i2c->adap.algo_data = i2c;
	i2c->adap.dev.parent = &pdev->dev;
	i2c->pctrl = devm_pinctrl_get_select_default(i2c->dev);

	ret = clk_prepare_enable(i2c->clk);

	ret = s3c24xx_i2c_init(i2c);
	clk_disable(i2c->clk);

	if (!(i2c->quirks & QUIRK_POLL)) {
		i2c->irq = platform_get_irq(pdev, 0);

		ret = devm_request_irq(&pdev->dev, i2c->irq, s3c24xx_i2c_irq,
				       0, dev_name(&pdev->dev), i2c);
	}

	ret = s3c24xx_i2c_register_cpufreq(i2c);

	i2c->adap.nr = i2c->pdata->bus_num;
	i2c->adap.dev.of_node = pdev->dev.of_node;

	platform_set_drvdata(pdev, i2c);

	ret = i2c_add_numbered_adapter(&i2c->adap);

	return 0;
}
```
先初始化i2c控制器的I/O地址、中断号等硬件资源，实现适配器的数据传输方法，最后通过 [i2c_add_adapter()](#i2c_add_adapter())函数完成适配器的构建。



### i2c_algorithm

i2c_algorithm 对应i2c总线数据通信算法，i2c适配器需要 i2c_algorithm 中提供的通信函数来实现对I2C总线上数据的发送和接收等操作。

```c
struct i2c_algorithm {
    /* 作为主设备时的发送函数 */
	int (*master_xfer)(struct i2c_adapter *adap, struct i2c_msg *msgs,
			   int num);

    /* 作为从设备时的发送函数 */
	int (*smbus_xfer) (struct i2c_adapter *adap, u16 addr,
			   unsigned short flags, char read_write,
			   u8 command, int size, union i2c_smbus_data *data);

	u32 (*functionality) (struct i2c_adapter *);
};

struct i2c_msg {
	__u16 addr;	/* slave address			*/
	__u16 flags;
#define I2C_M_RD		0x0001	/* read data, from slave to master */
					/* I2C_M_RD is guaranteed to be 0x0001! */
#define I2C_M_TEN		0x0010	/* this is a ten bit chip address */
#define I2C_M_DMA_SAFE		0x0200	/* the buffer of this message is DMA safe */
					/* makes only sense in kernelspace */
					/* userspace buffers are copied anyway */
#define I2C_M_RECV_LEN		0x0400	/* length will be first received byte */
#define I2C_M_NO_RD_ACK		0x0800	/* if I2C_FUNC_PROTOCOL_MANGLING */
#define I2C_M_IGNORE_NAK	0x1000	/* if I2C_FUNC_PROTOCOL_MANGLING */
#define I2C_M_REV_DIR_ADDR	0x2000	/* if I2C_FUNC_PROTOCOL_MANGLING */
#define I2C_M_NOSTART		0x4000	/* if I2C_FUNC_NOSTART */
#define I2C_M_STOP		0x8000	/* if I2C_FUNC_PROTOCOL_MANGLING */
	__u16 len;		/* msg length				*/
	__u8 *buf;		/* pointer to msg data			*/
};
```

```c
static int s3c24xx_i2c_xfer(struct i2c_adapter *adap,
			struct i2c_msg *msgs, int num)
{
	struct s3c24xx_i2c *i2c = (struct s3c24xx_i2c *)adap->algo_data;
	int retry;
	int ret;

	ret = clk_enable(i2c->clk);

	for (retry = 0; retry < adap->retries; retry++) {

		ret = s3c24xx_i2c_doxfer(i2c, msgs, num);

		udelay(100);
	}

	clk_disable(i2c->clk);
	return -EREMOTEIO;
}

static u32 s3c24xx_i2c_func(struct i2c_adapter *adap)
{
	return I2C_FUNC_I2C | I2C_FUNC_SMBUS_EMUL | I2C_FUNC_NOSTART |
		I2C_FUNC_PROTOCOL_MANGLING;
}

static const struct i2c_algorithm s3c24xx_i2c_algorithm = {
	.master_xfer		= s3c24xx_i2c_xfer,
	.functionality		= s3c24xx_i2c_func,
};
```
i2c_algorithm 中的master_xfer()函数通过操作寄存器来完成i2c_msg数据的传输。functionality()函数返回支持的通信协议。

s3c24xx_i2c_xfer()函数调用 s3c24xx_i2c_doxfer()函数传输i2c消息，s3c24xx_i2c_doxfer()先将适配器设置为主设备，使能中断，并调用 s3c24xx_i2c_message_start()函数传输开始信号和从设备地址。然后在中断处理函数 s3c24xx_i2c_irq()中，会根据 i2c状态寄存器的状态进行读写和停止操作。




## i2c 设备驱动

I2C设备驱动主要维护两个结构体：i2c_driver和i2c_client，实现和用户交互的文件操作集合fops、cdev等。此部分代码就是驱动开发者需要完成的。

### i2c_driver

i2c_driver 对应一套驱动方法，实现设备与总线的挂接，其主要函数是attach_adapter()和detach_client()。i2c_driver与i2c_client的关系是一对多，一个i2c_driver上可以支持多个同等类型的i2c_client。

```c
struct i2c_driver {
	unsigned int class;

	int (*probe)(struct i2c_client *, const struct i2c_device_id *);
	int (*remove)(struct i2c_client *);

	int (*probe_new)(struct i2c_client *);

	void (*shutdown)(struct i2c_client *);

	void (*alert)(struct i2c_client *, enum i2c_alert_protocol protocol,
		      unsigned int data);

	int (*command)(struct i2c_client *client, unsigned int cmd, void *arg);

	struct device_driver driver;
	const struct i2c_device_id *id_table;

	int (*detect)(struct i2c_client *, struct i2c_board_info *);
	const unsigned short *address_list;
	struct list_head clients;

	bool disable_i2c_core_irq_mapping;
};
```



### 注册i2c设备驱动

```c
struct i2c_device_id {
	char name[I2C_NAME_SIZE];
	kernel_ulong_t driver_data;	/* Data private to the driver */
};

static const struct i2c_device_id at24_ids[] = {
	{ "24c00",	0 },
	{ "24c01",	0},
	{ /* END OF LIST */ },
};

struct of_device_id {
	char	name[32];
	char	type[32];
	char	compatible[128];
	const void *data;
};

static const struct of_device_id at24_of_match[] = {
	{ .compatible = "atmel,24c00"},
	{ .compatible = "atmel,24c01"},
	{ /* END OF LIST */ },
};

static struct i2c_driver at24_driver = {
	.driver = {
		.name = "at24",
		.of_match_table = at24_of_match,　　//设备树匹配方式
		.acpi_match_table = ACPI_PTR(at24_acpi_ids),
	},
	.probe_new = at24_probe,
	.remove = at24_remove,
	.id_table = at24_ids,  //匹配ID方式
};

i2c_add_driver(&at24_driver);
```
实例一个 i2c_driver，然后通过[i2c_add_driver()](#i2c_add_driver()) 函数注册i2c设备驱动。id_table数组元素中的名称指明了支持的I2C设备。



### i2c_client

i2c_client 对应真实的i2c物理设备，描述了i2c设备的硬件信息，设备挂接在i2c适配器上，通过i2c适配器与CPU交换数据。所有的i2c设备都在/sys/bus/i2c/目录中显示，以适配器地址和芯片地址的形式列出。

```c
struct i2c_client {
	unsigned short flags;		/* div., see below		*/
	unsigned short addr;		/* chip address - NOTE: 7bit	*/
					/* addresses are stored in the	*/
					/* _LOWER_ 7 bits		*/
	char name[I2C_NAME_SIZE];
	struct i2c_adapter *adapter;	/* the adapter we sit on	*/
	struct device dev;		/* the device structure		*/
	int init_irq;			/* irq set at initialization	*/
	int irq;			/* irq issued by device		*/
	struct list_head detected;
};
```



### 注册i2c设备

#### 在板文件注册

i2c_client 信息通常在BSP板文件中通过 i2c_board_info 填充。注册I2C设备时，先获取相应总线号上的适配器，然后通过[i2c_new_device()](#i2c_new_device())函数生成一个i2c_client。

```c
static struct i2c_board_info  my_i2c_dev_info = {
	I2C_BOARD_INFO("my_i2c_dev", 0x20),  //名字，设备地址
};

	adapter = i2c_get_adapter(bus);

    client = i2c_new_device(adapter, my_i2c_dev_info);
```

也可以通过 i2c_register_board_info() 函数来注册。
```c
/*
 * busnum：那条总线
 * info：i2c设备信息数组
 * len：数组有几项
 */
int i2c_register_board_info(int busnum, struct i2c_board_info const *info, unsigned len);
```



#### 在用户空间注册

```shell
#注册设备
echo my_i2c_dev 0x55 > sys/devices/platform/s3c2440-i2c.0/i2c-0/new_device

#注销设备
echo 0x55 > sys/devices/platform/s3c2440-i2c.0/i2c-0/delete_device
```




## i2c-dev.c
```c
static int __init i2c_dev_init(void)
{
	res = register_chrdev_region(MKDEV(I2C_MAJOR, 0), I2C_MINORS, "i2c");

	i2c_dev_class = class_create(THIS_MODULE, "i2c-dev");

	i2c_dev_class->dev_groups = i2c_groups;

	res = bus_register_notifier(&i2c_bus_type, &i2cdev_notifier);

	i2c_for_each_dev(NULL, i2cdev_attach_adapter);

	return 0;
}

static const struct file_operations i2cdev_fops = {
	.owner		= THIS_MODULE,
	.llseek		= no_llseek,
	.read		= i2cdev_read,
	.write		= i2cdev_write,
	.unlocked_ioctl	= i2cdev_ioctl,
	.compat_ioctl	= compat_i2cdev_ioctl,
	.open		= i2cdev_open,
	.release	= i2cdev_release,
};

static int i2cdev_attach_adapter(struct device *dev, void *dummy)
{
	struct i2c_adapter *adap;
	struct i2c_dev *i2c_dev;
	int res;

	adap = to_i2c_adapter(dev);

	i2c_dev = get_free_i2c_dev(adap);
	if (IS_ERR(i2c_dev))
		return PTR_ERR(i2c_dev);

	cdev_init(&i2c_dev->cdev, &i2cdev_fops);
	i2c_dev->cdev.owner = THIS_MODULE;
	res = cdev_add(&i2c_dev->cdev, MKDEV(I2C_MAJOR, adap->nr), 1);

	i2c_dev->dev = device_create(i2c_dev_class, &adap->dev,
				     MKDEV(I2C_MAJOR, adap->nr), NULL,
				     "i2c-%d", adap->nr);

	return 0;
}
```

实现了i2c适配器设备文件的功能，每一个i2c适配器都被分配一个设备。通过适配器访问设备时的主设备号都为 89，次设备号为 0～255。应用程序通过“i2c-%d”设备结点并使用文件操作接口 open()、write()、read()、ioctl()和 close()等来访问这个设备。



## i2c-tools

下载地址：https://mirrors.edge.kernel.org/pub/software/utils/i2c-tools/



