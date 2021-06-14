<!--
 * @Description: platform
 * @Author: luo_u
 * @Date: 2020-06-01 10:47:40
 * @LastEditTime: 2020-10-02 12:51:27
--> 

[TOC]

platform 是一种虚拟的总线，总线将设备和驱动绑定。在系统每注册一个设备的时候，会寻找与之匹配的驱动；相反的，在系统每注册一个驱动的时候，会寻找与之匹配的设备，而匹配由总线完成。相应的设备称为 platform_device，而驱动成为 platform_driver。

platform 使设备被挂接在一个总线上，隔离 BSP 和驱动。在 BSP 中定义 platform 设备和设备使用的资源、设备的具体配置信息，而在驱动中只需要通过 API 去获取资源和数据，做到了板相关代码和驱动代码的分离，使得驱动具有更好的可扩展性和跨平台性。



## platform_device

```c
struct platform_device {
	const char	* name;  
	int		id;
	struct device	dev;
	u32		num_resources;
	struct resource	* resource;
	const struct platform_device_id	*id_entry;
	struct mfd_cell *mfd_cell;
	struct pdev_archdata	archdata;
};

int platform_add_devices(struct platform_device **devs, int num);
```

platform_device 的定义在arch/arm/mach-xxx/目录下的BSP板文件中实现，将 platform_device 归纳为一个数组，最终通过 platform_add_devices()函数统一注册。



### platform_device_register()
```c
int platform_device_register(struct platform_device *pdev)
{
	device_initialize(&pdev->dev);
	return platform_device_add(pdev);
}

int platform_device_add(struct platform_device *pdev)
{
	if (!pdev->dev.parent)
		pdev->dev.parent = &platform_bus;

	pdev->dev.bus = &platform_bus_type;

	for (i = 0; i < pdev->num_resources; i++) {
		struct resource *p, *r = &pdev->resource[i];

		if (r->name == NULL)
			r->name = dev_name(&pdev->dev);

		p = r->parent;
		if (!p) {
			if (resource_type(r) == IORESOURCE_MEM)
				p = &iomem_resource;
			else if (resource_type(r) == IORESOURCE_IO)
				p = &ioport_resource;
		}
	}

	device_add(&pdev->dev);
}

int device_add(struct device *dev)
{
	dev = get_device(dev);

	bus_add_device(dev);

	bus_probe_device(dev);
}

void bus_probe_device(struct device *dev)
{
	if (bus && bus->p->drivers_autoprobe) {
		ret = device_attach(dev);
	}
}

int device_attach(struct device *dev)
{
	if (dev->driver) {
		klist_node_attached(&dev->p->knode_driver);

		ret = device_bind_driver(dev);

	} else {
		ret = bus_for_each_drv(dev->bus, NULL, dev, __device_attach);
	}
}

static int __device_attach(struct device_driver *drv, void *data)
{
	struct device *dev = data;

	if (!driver_match_device(drv, dev))
		return 0;

	return driver_probe_device(drv, dev);
}

static inline int driver_match_device(struct device_driver *drv,
				      struct device *dev)
{
	return drv->bus->match ? drv->bus->match(dev, drv) : 1;
}
```
注册 platform_device 后，会遍历 platform_bus 中的 platform_driver，根据 platform_bus 的 match 函数规则来匹配相应的平台驱动。

```c
static struct platform_device hello_device=
{
    .name = "hello_platform",
    .id = -1,
    .resource = hello_resource,
    .num_resources = ARRAY_SIZE(hello_resource),
    .dev = 
    {
    	.platform_data = &plat_data,
    	.release = hello_release,
    },
};

static int hello_init(void)
{
	platform_device_register(&hello_device);

	return 0;
}
 
static void hello_exit(void)
{
	platform_device_unregister(&hello_device);
	return;
}
```

device 结构体成员 platform_data 用来存放设备的私有数据，数据内容结构体可自定义，驱动中可通过设备指针来获取。
```c
struct xxx_plat_data *pdata = pdev->dev.platform_data;
```



## platform_driver

```c
struct platform_driver {
	int (*probe)(struct platform_device *);
	int (*remove)(struct platform_device *);
	void (*shutdown)(struct platform_device *);
	int (*suspend)(struct platform_device *, pm_message_t state);
	int (*resume)(struct platform_device *);
	struct device_driver driver;
	const struct platform_device_id *id_table;
};

/* platform_driver 的注册与注销 */
int platform_driver_register(struct platform_driver *drv);
void platform_driver_unregister(struct platform_driver *drv);
```


### platform_driver_register()
```c
int platform_driver_register(struct platform_driver *drv)
{
	drv->driver.bus = &platform_bus_type;
	if (drv->probe)
		drv->driver.probe = platform_drv_probe;
	if (drv->remove)
		drv->driver.remove = platform_drv_remove;
	if (drv->shutdown)
		drv->driver.shutdown = platform_drv_shutdown;

	return driver_register(&drv->driver);
}

int driver_register(struct device_driver *drv)
{
	struct device_driver *other;

	other = driver_find(drv->name, drv->bus);
	if (other) {
		put_driver(other);
		printk(KERN_ERR "Error: Driver '%s' is already registered, "
			"aborting...\n", drv->name);
		return -EBUSY;
	}

	bus_add_driver(drv);
}

int bus_add_driver(struct device_driver *drv)
{
	if (drv->bus->p->drivers_autoprobe) {
		error = driver_attach(drv);
	}
}

int driver_attach(struct device_driver *drv)
{
	return bus_for_each_dev(drv->bus, NULL, drv, __driver_attach);
}

static int __driver_attach(struct device *dev, void *data)
{
	struct device_driver *drv = data;

	if (!driver_match_device(drv, dev))
		return 0;
}

static inline int driver_match_device(struct device_driver *drv,
				      struct device *dev)
{
	return drv->bus->match ? drv->bus->match(dev, drv) : 1;
}
```
当注册 platform_driver 后，会遍历 platform_bus 中的 platform_device ，根据 platform_bus 的 match 函数规则来匹配相应的平台设备。



## platform 总线

```c
struct bus_type platform_bus_type = {
	.name		= "platform",
	.dev_attrs	= platform_dev_attrs,
	.match		= platform_match,
	.uevent		= platform_uevent,
	.pm		= &platform_dev_pm_ops,
};

typedef unsigned long kernel_ulong_t;

struct platform_device_id {
	char name[PLATFORM_NAME_SIZE];
	kernel_ulong_t driver_data;
};
```

match()成员函数确定了 platform_device 和 platform_driver 之间的匹配规则。主要是看两者的name字段是否相同。
```c
static int platform_match(struct device *dev, struct device_driver *drv)
{
	struct platform_device *pdev = to_platform_device(dev);
	struct platform_driver *pdrv = to_platform_driver(drv);

	/* When driver_override is set, only bind to the matching driver */
	if (pdev->driver_override)
		return !strcmp(pdev->driver_override, drv->name);

	/* Attempt an OF style match first */
	if (of_driver_match_device(dev, drv))
		return 1;

	/* Then try ACPI style match */
	if (acpi_driver_match_device(dev, drv))
		return 1;

	/* Then try to match against the id table */
	if (pdrv->id_table)
		return platform_match_id(pdrv->id_table, pdev) != NULL;

	/* fall-back to driver name match */
	return (strcmp(pdev->name, drv->name) == 0);
}
```



## platform 设备资源

```c
struct resource {
	resource_size_t start;  /* 资源的起始物理地址 */
	resource_size_t end;    /* 资源的结束物理地址 */
	const char *name;
	unsigned long flags;  /* 资源的类型 */
	struct resource *parent, *sibling, *child;
};

struct resource *platform_get_resource(struct platform_device *dev,
				       unsigned int type, unsigned int num)
```
资源类型flags取値：
- IORESOURCE_IO
- IORESOURCE_MEM  platform_device 占据的内存
- IORESOURCE_IRQ  platform_device 使用的中断号
- IORESOURCE_DMA

当 flags 为 IORESOURCE_MEM 时，start、end 分别表示该 platform_device 占据的内存的开始地址和结束地址;
当 flags 为 IORESOURCE_IRQ 时，start、end 分别表示该 platform_device 使用的中断号的开始值和结束值，如果只使用了 1 个中断号，开
始和结束值相同。

platform 设备资源的定义通常在 BSP 的板文件中进行，在驱动中通过 platform_get_resource()函数获取设备资源。例如 DM9000 网卡的资源定义：

```c
static struct resource dm9000_resource[] = {
	[0] = {
		.start = 0x18000000,
		.end = 0x18000000 + 3,
		.flags = IORESOURCE_MEM
	},

	[1] = {
		.start = 0x18000000 + 0x4,
		.end = 0x18000000 + 0x7,
		.flags = IORESOURCE_MEM
	},

	[2] = {
		.start = IRQ_EINT(7),
		.end = IRQ_EINT(7),
		.flags = IORESOURCE_IRQ,
	}
};
```

在驱动中攻取这 3 份资源:
```c
db->addr_res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
db->data_res = platform_get_resource(pdev, IORESOURCE_MEM, 1);
db->irq_res = platform_get_resource(pdev, IORESOURCE_IRQ, 0);
```

