<!--
 * @Description: sysfs
 * @Author: luo_u
 * @Date: 2020-07-08 21:30:23
 * @LastEditTime: 2020-10-02 11:14:41
--> 

[TOC]

## sysfs

Linux 2.6 的内核引入了 sysfs 文件系统，sysfs 是一个虚拟的文件系统，是内核对象(kobject)、属性(kobj_type)及它们相互关系的一种表现机制。把连接在系统上的设备和总线组织成为一个分级的文件，它们可以由用户空间存取，向用户空间导出内核数据结构以及它们的属性，以展示设备驱动模型中各组件的层次关系。
```
/sys/
├── block  	包含系统中发现的块设备;
├── bus   	包含内核注册的总线;
├── class 	包含注册到内核的设备类;
├── bus
│   ├── i2c
│ 	│ 	├── drivers
│   │ 	└── devices   包含总线下的所有设备
│	│ 		├──i2c-0 -> ../../../devices/pci0000:00/0000:00:02.0/i2c-0/
│	│    	└──i2c-1 -> ../../../devices/pci0000:00/0000:00:02.0/i2c-1/
│   ├── pci
│   ├── spi
│   ├── usb
├── class
│   ├── block
│   ├── bluetooth
│   ├── dma
│   ├── gpio
│   ├── i2c-adapter
│   ├── i2c-dev
│   ├── input
│   ├── misc
│   ├── phy
│   ├── pwm
│   ├── rtc
│   ├── tty
├── dev
│   ├── block
│   └── char
├── devices   含系统所有的设备，并根据设备挂接的总线类型组织成层次结构;
├── firmware
│   ├── acpi
│   ├── dmi
│   └── memmap
├── fs
├── hypervisor
├── kernel
├── module
└── power
```





## kobject

总线、驱动和设备都对应sysfs 中的一个目录，实际上都是kobject 的派生类(device 结构体直接包含了 kobject kobj 成员，而 bus_type和 device_driver 则透过 subsys_private、driver_private 间接包含 kobject)，每一个在内核注册的kobject对象都对应 sysfs中的一个目录。

```c
struct kobject {
	const char		*name;
	struct list_head	entry;
	struct kobject		*parent;
	struct kset		*kset;
	struct kobj_type	*ktype;
	struct sysfs_dirent	*sd;
	struct kref		kref;
	unsigned int state_initialized:1;
	unsigned int state_in_sysfs:1;
	unsigned int state_add_uevent_sent:1;
	unsigned int state_remove_uevent_sent:1;
	unsigned int uevent_suppress:1;
};

struct kobj_type {
	void (*release)(struct kobject *kobj);
	const struct sysfs_ops *sysfs_ops;
	struct attribute **default_attrs;   /* 属性数组 */
	const struct kobj_ns_type_operations *(*child_ns_type)(struct kobject *kobj);
	const void *(*namespace)(struct kobject *kobj);
	void (*get_ownership)(struct kobject *kobj, kuid_t *uid, kgid_t *gid);
};

struct sysfs_ops {
	ssize_t	(*show)(struct kobject *, struct attribute *, char *);
	ssize_t	(*store)(struct kobject *, struct attribute *, const char *, size_t);
};

int kobject_init_and_add(struct kobject *kobj, struct kobj_type *ktype,
			 struct kobject *parent, const char *fmt, ...)

int kobject_add(struct kobject *kobj, struct kobject *parent,
		const char *fmt, ...)

void kobject_init(struct kobject *kobj, struct kobj_type *ktype)

void kobject_del(struct kobject *kobj)

struct kobject *kobject_get(struct kobject *kobj)
int kobject_set_name(struct kobject *kobj, const char *fmt, ...)
```

kobj_type 是 kobject 的属性，sysfs_ops 表示对属性的操作函数，包含一个读函数和一个写函数。release()函数在 kobject 的引用计数为0时会被调用。



### attribute

总线、设备和驱动中的各个attribute 对应sysfs 中的一个文件，attribute 会伴随着show()
和store()这两个函数，分别用于读和写该 attribute 对应的 sysfs 文件结点。

udev 规则中各信息的来源实际上就是 bus_type、device_driver、device 以及 attribute 等所对应 sysfs 节点。

```
struct attribute {
	const char		*name;
	umode_t			mode;
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	bool			ignore_lockdep:1;
	struct lock_class_key	*key;
	struct lock_class_key	skey;
#endif
};
```


## kset

kset 是具有相同类型的 kobject 集合。

```c
struct kset {
	struct list_head list;
	spinlock_t list_lock;
	struct kobject kobj;
	const struct kset_uevent_ops *uevent_ops;  /* 热插拔事件 */
}

struct kset_uevent_ops {
	int (* const filter)(struct kset *kset, struct kobject *kobj);   /* 事件过滤，如果返回 0，将不传递事件到用户空间 */
	const char *(* const name)(struct kset *kset, struct kobject *kobj);  /* 返回子系统的名字 */
	int (* const uevent)(struct kset *kset, struct kobject *kobj,
		      struct kobj_uevent_env *env);  /* 将用户空间需要的参数添加到环境变量中 */
};

int kset_register(struct kset *k)

void kset_unregister(struct kset *k)
```

当驱动程序将 kobject 注册到设备驱动模型时，也就是调用 kobject_add()和 kobject_del()函数时，会产生热插拔事件，内核会根据 kobject 的 kset 指针找到所属的 kset 结构体，执行 uevent_ops 中的热插拔函数。



## bus

Linux内核分别使用 bus_type、device_driver 和 device 来描述总线、驱动和设备，这三个结构体定义于 include/linux/device.h 头文件中。

```c
struct bus_type {
	const char		*name;
	struct bus_attribute	*bus_attrs;
	struct device_attribute	*dev_attrs;
	struct driver_attribute	*drv_attrs;

	int (*match)(struct device *dev, struct device_driver *drv);
	int (*uevent)(struct device *dev, struct kobj_uevent_env *env);
	int (*probe)(struct device *dev);
	int (*remove)(struct device *dev);
	void (*shutdown)(struct device *dev);
	int (*suspend)(struct device *dev, pm_message_t state);
	int (*resume)(struct device *dev);

	const struct dev_pm_ops *pm;
	struct subsys_private *p;
};
```
device_driver 和 device 分别表示驱动和设备，这两者都必须依附于一种总线，因此都包含struct bus_type 指针。设备和驱动是分开注册的，由 bus_type 的 match()成员函数将两者捆绑在一起。


### bus_register()
```c
int bus_register(struct bus_type *bus)
{
	int retval;
	struct subsys_private *priv;
	struct lock_class_key *key = &bus->lock_key;

	priv = kzalloc(sizeof(struct subsys_private), GFP_KERNEL);

	priv->bus = bus;
	bus->p = priv;

	BLOCKING_INIT_NOTIFIER_HEAD(&priv->bus_notifier);

	retval = kobject_set_name(&priv->subsys.kobj, "%s", bus->name);

	priv->subsys.kobj.kset = bus_kset;
	priv->subsys.kobj.ktype = &bus_ktype;
	priv->drivers_autoprobe = 1;

	retval = kset_register(&priv->subsys);

	retval = bus_create_file(bus, &bus_attr_uevent);

	priv->devices_kset = kset_create_and_add("devices", NULL,
						 &priv->subsys.kobj);
	if (!priv->devices_kset) {
		retval = -ENOMEM;
	}

	priv->drivers_kset = kset_create_and_add("drivers", NULL,
						 &priv->subsys.kobj);
	if (!priv->drivers_kset) {
		retval = -ENOMEM;
	}

	INIT_LIST_HEAD(&priv->interfaces);
	__mutex_init(&priv->mutex, "subsys mutex", key);
	klist_init(&priv->klist_devices, klist_devices_get, klist_devices_put);
	klist_init(&priv->klist_drivers, NULL, NULL);

	retval = add_probe_files(bus);

	retval = bus_add_groups(bus, bus->bus_groups);
}

void bus_unregister(struct bus_type *bus)
```
bus_register()函数用来注册总线，可在/sys/bus目录下看到。



### bus_attribute
```c
struct bus_attribute {
	struct attribute	attr;
	ssize_t (*show)(struct bus_type *bus, char *buf);
	ssize_t (*store)(struct bus_type *bus, const char *buf, size_t count);
};

#define BUS_ATTR(_name, _mode, _show, _store)	\
	struct bus_attribute bus_attr_##_name = __ATTR(_name, _mode, _show, _store)

int bus_create_file(struct bus_type *bus, struct bus_attribute *attr)
{
	int error;
	if (bus_get(bus)) {
		error = sysfs_create_file(&bus->p->subsys.kobj, &attr->attr);
		bus_put(bus);
	} else
		error = -EINVAL;
	return error;
}
```
可用BUS_ATTR宏来初始化一个 bus_attribute 结构体。创建总线的属性，需要调用 bus_create_file() 函数。
```c
static ssize_t show_bus_version(struct bus_type *bus, char *buf)
{
	return snprintf(buf, PAGE_SIZE, "%s\n", "revision: 1.0");
}

static BUS_ATTR(version, S_IRUGO, show_bus_version, NULL);

bus_create_file(&my_bus_type, &bus_attr_version)
```



## device

```c
struct device {
	struct device		*parent;
	struct device_private	*p;
	struct kobject kobj;
	const char		*init_name; /* initial name of the device */
	const struct device_type *type;
	struct mutex		mutex;	/* mutex to synchronize calls to driver */
	struct bus_type	*bus;		/* type of bus device is on */
	struct device_driver *driver;	/* which driver has allocated this device */
	void		*platform_data;	/* Platform specific data, device core doesn't touch it */
	struct dev_pm_info	power;
	struct dev_power_domain	*pwr_domain;

#ifdef CONFIG_NUMA
	int		numa_node;	/* NUMA node this device is close to */
#endif
	u64		*dma_mask;	/* dma mask (if dma'able device) */
	u64		coherent_dma_mask;/* Like dma_mask, but for alloc_coherent mappings as
					     not all hardware supports 64 bit addresses for consistent
					     allocations such descriptors. */

	struct device_dma_parameters *dma_parms;
	struct list_head	dma_pools;	/* dma pools (if dma'ble) */
	struct dma_coherent_mem	*dma_mem; /* internal for coherent mem override */
	struct dev_archdata	archdata;
	struct device_node	*of_node; /* associated device tree node */
	dev_t			devt;	/* dev_t, creates the sysfs "dev" */
	spinlock_t		devres_lock;
	struct list_head	devres_head;
	struct klist_node	knode_class;
	struct class		*class;
	const struct attribute_group **groups;	/* optional groups */
	void	(*release)(struct device *dev);
};
```


### device_register()
```c
int device_register(struct device *dev)
{
	device_initialize(dev);
	return device_add(dev);
}

void device_initialize(struct device *dev)
{
	dev->kobj.kset = devices_kset;
	kobject_init(&dev->kobj, &device_ktype);
	INIT_LIST_HEAD(&dev->dma_pools);
	mutex_init(&dev->mutex);
	lockdep_set_novalidate_class(&dev->mutex);
	spin_lock_init(&dev->devres_lock);
	INIT_LIST_HEAD(&dev->devres_head);
	device_pm_init(dev);
	set_dev_node(dev, -1);
#ifdef CONFIG_GENERIC_MSI_IRQ
	INIT_LIST_HEAD(&dev->msi_list);
#endif
	INIT_LIST_HEAD(&dev->links.consumers);
	INIT_LIST_HEAD(&dev->links.suppliers);
	dev->links.status = DL_DEV_NO_DRIVER;
}

int device_add(struct device *dev)
{
	struct device *parent;
	struct kobject *kobj;
	struct class_interface *class_intf;
	int error = -EINVAL;
	struct kobject *glue_dir = NULL;

	dev = get_device(dev);

	if (!dev->p) {
		error = device_private_init(dev);
	}

	if (dev->init_name) {
		dev_set_name(dev, "%s", dev->init_name);
		dev->init_name = NULL;
	}

	/* subsystems can specify simple device enumeration */
	if (!dev_name(dev) && dev->bus && dev->bus->dev_name)
		dev_set_name(dev, "%s%u", dev->bus->dev_name, dev->id);

	if (!dev_name(dev)) {
		error = -EINVAL;
		goto name_error;
	}

	parent = get_device(dev->parent);
	kobj = get_device_parent(dev, parent);
	if (kobj)
		dev->kobj.parent = kobj;

	/* use parent numa_node */
	if (parent && (dev_to_node(dev) == NUMA_NO_NODE))
		set_dev_node(dev, dev_to_node(parent));

	error = kobject_add(&dev->kobj, dev->kobj.parent, NULL);

	/* notify platform of device entry */
	error = device_platform_notify(dev, KOBJ_ADD);

	error = device_create_file(dev, &dev_attr_uevent);

	error = device_add_class_symlinks(dev);
	error = device_add_attrs(dev);
	error = bus_add_device(dev);
	error = dpm_sysfs_add(dev);
	device_pm_add(dev);

	if (MAJOR(dev->devt)) {
		error = device_create_file(dev, &dev_attr_dev);

		error = device_create_sys_dev_entry(dev);

		devtmpfs_create_node(dev);
	}

	if (dev->bus)
		blocking_notifier_call_chain(&dev->bus->p->bus_notifier,
					     BUS_NOTIFY_ADD_DEVICE, dev);

	kobject_uevent(&dev->kobj, KOBJ_ADD);
	bus_probe_device(dev);
	if (parent)
		klist_add_tail(&dev->p->knode_parent,
			       &parent->p->klist_children);

	if (dev->class) {
		mutex_lock(&dev->class->p->mutex);
		/* tie the class to the device */
		klist_add_tail(&dev->knode_class,
			       &dev->class->p->klist_devices);

		/* notify any interfaces that the device is here */
		list_for_each_entry(class_intf,
				    &dev->class->p->interfaces, node)
			if (class_intf->add_dev)
				class_intf->add_dev(dev, class_intf);
		mutex_unlock(&dev->class->p->mutex);
	}
}
```

```c
struct device_attribute {
	struct attribute	attr;
	ssize_t (*show)(struct device *dev, struct device_attribute *attr,
			char *buf);
	ssize_t (*store)(struct device *dev, struct device_attribute *attr,
			 const char *buf, size_t count);
};

#define DEVICE_ATTR(_name, _mode, _show, _store) \
	struct device_attribute dev_attr_##_name = __ATTR(_name, _mode, _show, _store)

int device_create_file(struct device *dev,
		       const struct device_attribute *attr)
```



## device_driver

```c
struct device_driver {
	const char		*name;
	struct bus_type		*bus;
	struct module		*owner;
	const char		*mod_name;	/* used for built-in modules */
	bool suppress_bind_attrs;	/* disables bind/unbind via sysfs */
	const struct of_device_id	*of_match_table;

	int (*probe) (struct device *dev);
	int (*remove) (struct device *dev);
	void (*shutdown) (struct device *dev);
	int (*suspend) (struct device *dev, pm_message_t state);
	int (*resume) (struct device *dev);
	const struct attribute_group **groups;

	const struct dev_pm_ops *pm;
	struct driver_private *p;
};
```


### driver_register()
```c
int driver_register(struct device_driver *drv)
{
	int ret;
	struct device_driver *other;

	if ((drv->bus->probe && drv->probe) ||
	    (drv->bus->remove && drv->remove) ||
	    (drv->bus->shutdown && drv->shutdown))
		printk(KERN_WARNING "Driver '%s' needs updating - please use "
			"bus_type methods\n", drv->name);

	other = driver_find(drv->name, drv->bus);

	ret = bus_add_driver(drv);

	ret = driver_add_groups(drv, drv->groups);

	kobject_uevent(&drv->p->kobj, KOBJ_ADD);
}
```


```c
struct driver_attribute {
	struct attribute attr;
	ssize_t (*show)(struct device_driver *driver, char *buf);
	ssize_t (*store)(struct device_driver *driver, const char *buf,
			 size_t count);
};

#define DRIVER_ATTR(_name, _mode, _show, _store)	\
struct driver_attribute driver_attr_##_name =		\
	__ATTR(_name, _mode, _show, _store)

int driver_create_file(struct device_driver *drv,
		       const struct driver_attribute *attr)
```


