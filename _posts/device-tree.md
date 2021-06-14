
[TOC]

设备树可以描述的信息包括CPU的数量和类别、内存基地址和大小、总线和桥、外设连接、中断控制器和中断使用情况、GPIO控制器和GPIO使用情况、Clock控制器和Clock使用情况。另外，设备树对于可热插拔的热备不进行具体描述，它只描述用于控制该热插拔设备的控制器。对于同一SOC的不同主板，只需更换设备树文件.dtb即可实现不同主板的无差异支持，而无需更换内核文件。

设备树中的设备树节点在文件系统中有与之对应的文件，位于/proc/device-tree目录。在这里子节点是一个文件夹，属性是一个文件。设备节点属性的设置方法可参考内核文档Documentation/devicetree。

https://www.devicetree.org/


设备树包含DTC（device tree compiler），DTS（device tree source和DTB（device tree blob）。



## DTS

dts文件是一种ASCII文本对Device Tree的描述，放置在内核的/arch/arm/boot/dts目录。一般而言，一个dts文件对应一个ARM的板子。

由于一个SOC可能有多个不同的电路板，而每个电路板拥有一个dts文件。这些dts势必会存在许多共同部分，为了减少代码的冗余，设备树将这些共同部分提炼保存在dtsi文件中，供不同的dts共同使用，使用时需要包含dtsi文件。

```
#include "exynos4.dtsi"
```

device tree的基本单元是node。这些node被组织成树状结构，除了root node，每个node都只有一个parent。一个dts文件中只能有一个root node。

每一个“{}”都是一个设备树节点，“{}”中包含的内容是节点属性，属性的定义采用property ＝ value的形式，value有三种情况：

-  字符串或字符串数组，用双引号表示。
- u32整数，用尖括号表示。
- 二进制数，用方括号表示。

设备树中的每个节点都按照以下节点基本格式：
```
[label:] node-name[@unit-address] { 
   [properties definitions] 
   [child nodes] 
}
```

- label定义一个标签，是可选的，在标签名前加“&”符号就可引用该节点。

- node-name 用于标识节点的名称，root node的节点名必须是“/”。

- `@unit-address` ，其中“@”符号是分割符，“unit-address”是单元地址， 它的值要和节点`reg`属性的第一个地址一致。如果节点没有`reg`属性，可以直接省略， 不过这时要求同级别的设备树下节点名唯一。同级别的子节点的节点名可以相同，但是要求地址不同，node-name@unit-address 的整体要求同级唯一。

子节点前面加一个“&”符号， 这表示该节点在向已经存在的子节点追加数据。




### 节点

#### chosen

chosen 节点主要用来描述由系统firmware指定的runtime parameter。如果存在chosen node，其parent node必须是根节点。可以用作uboot向linux内核传递配置参数，如：`bootargs` ,`console`。

```
/ {
	chosen {
		bootargs = "root=/dev/mmcblk0p2 rw rootfstype=ext4 rootdelay=1 rootwait";
		stdout-path = "serial2:115200n8";
	};
};  
```

#### aliases

aliases 节点定义了一个别名。

```
aliases {
    spi0 = &spi_0;
    i2c0 = &i2c_0;
    serial0 = &serial_0;
};
```

#### memory

memory node是所有设备树文件的必备节点，它定义了系统物理内存的layout。对于memory node，其`device_type`属性必须等于`memory`。`reg`属性定义了memory的起始地址和长度。

```
memory@40000000 {
    device_type = "memory";
    reg = <0x40000000 0x40000000>;
};
```



#### interrupt-controller

```
gic: interrupt-controller@10490000 {
    #interrupt-cells = <3>;
    interrupt-controller;
};
```

```
wakup_eint: wakeup-interrupt-controller {
    interrupt-parent = <&gic>;
    interrupts = <GIC_SPI 32 IRQ_TYPE_LEVEL_HIGH>;
};
```


- `interrupt-controller` 是一个空属性，用来声明这个node是一个中断控制器。
- `#interrupt-cells`标识这个控制器需要几个单位做中断描述符，描述子节点中`interrupts`属性使用了父节点中`interrupts`属性的具体的哪个值。如果父节点的属性値是3，则子节点`interrupts`属性中的3个值分别代表<中断域 中断 触发方式>；如果父节点的属性値是2，则是<中断 触发方式>。
- `interrupt-parent`标识此设备节点属于哪一个中断控制器，如果没有设置这个属性，会自动依附父节点的。
- `interrupts`标识一个中断。而具体每个cell的含义，由驱动的实现决定，也会在Device Tree的binding文档中说明。

#### gpio-controller

```
gpa0: gpa0 {
    gpio-controller;
    #gpio-cells = <2>;
};
```

- `gpio-controller`是一个空属性，用来说明该节点描述的是一个gpio控制器。
- `#gpio-cells`用来描述gpio使用节点的属性一个cell的内容，即 `属性 = <&引用gpio节点别名  gpio号  有效电平>



### 属性

#### compatible

```
leds {
    compatible = "gpio-leds";

    led2 {
        label = "red:system";
        gpios = <&gpl2 0 GPIO_ACTIVE_HIGH>;
        default-state = "on";
        linux,default-trigger = "heartbeat";
    };

    led3 {
        label = "red:user";
        gpios = <&gpk1 1 GPIO_ACTIVE_HIGH>;
        default-state = "on";
    };
};
```
compatible属性值由一个或多个字符串组成，有多个字符串时使用“,”分隔开。

设备树中的每个设备节点都有一个compatible属性，compatible属性用于驱动和设备的绑定。platform总线根据设备节点”compatible”属性和驱动程序中的of_match_table指针所指向的of_device_id结构里的compatible字段匹配的，只有dts里的compatible字段的名字和驱动程序中of_device_id里的compatible字段的名字一样，驱动程序才能进入probe函数。

```
static const struct of_device_id of_gpio_leds_match[] = {
	{ .compatible = "gpio-leds", },
	{},
};

static struct platform_driver gpio_led_driver = {
	.probe		= gpio_led_probe,
	.shutdown	= gpio_led_shutdown,
	.driver		= {
		.name	= "leds-gpio",
		.of_match_table = of_gpio_leds_match,
	},
};
```



#### model

model属性用于指定设备的制造商和型号，格式自定义。

```
model = "TOPEET iTop 4412 Elite board based on Exynos4412";
```



#### status

status属性用于指示设备的操作状态，可以禁止设备或者启用设备，默认情况下不设置status属性设备是使能的。

```
status = "disabled";
```

#### ranges

ranges属性为地址转换表，表明了子节点地址空间和父地址空间的映射方法，常见格式是`ranges = <子地址, 父地址, 转换长度>`。 如果父地址空间和子地址空间相同则无需转换，只写`renges,`内容为空，或直接省略renges属性。

比如对于#address-cells和#size-cells都为1的话，以ranges=<0x0 0x10 0x20>为例，表示将子地址的从0x0~(0x0 + 0x20)的地址空间映射到父地址的0x10~(0x10 + 0x20)。

#### name

name属性在旧的设备树中用于确定节点名， 现在已经弃用。

```
name = "name"
```

#### device_type

对于device node，device_type属性定义了该node的设备类型，如`cpu`、`serial`等。对于memory node，其device_type属性必须等于`memory`。

#### reg

reg用于描述地址的offset和length。该属性的值被解析成任意长度的（address，length）组合，其中address和size的値是在其parent node中的`#address-cells`和`#size-cells`属性定义的。

```
reg = <0x03830000 0x100>;
```

如果一个device node中包含了有寻址需求的sub node，那么就必须要定义`#address-cells`和`#size-cells`这两个属性。“#”是number的意思，这两个属性是用来描述sub node中的reg属性的地址域特性的，也就是说需要用多少个u32的cell来描述该地址域。

```
/ {
	#address-cells = <1>;
	#size-cells = <1>;
};
```



## 获取节点信息

内核提供了一组函数用于获取设备节点属性的函数，这些函数以of_开头，称为OF操作函数。 

### device_node

```c
struct device_node {
	const char *name;  /* 节点中属性为name的值 */
	phandle phandle;  /* 节点中属性为device_type的值 */
	const char *full_name;
	struct fwnode_handle fwnode;

	struct	property *properties;
	struct	property *deadprops;	/* removed properties */
	struct	device_node *parent;
	struct	device_node *child;
	struct	device_node *sibling;
#if defined(CONFIG_OF_KOBJ)
	struct	kobject kobj;
#endif
	unsigned long _flags;
	void	*data;
#if defined(CONFIG_SPARC)
	unsigned int unique_id;
	struct of_irq_controller *irq_trans;
#endif
};
```

### 查找节点

```c
/* 根据节点路径寻找 */
struct device_node *of_find_node_by_path(const char *path)

/* 根据节点名字寻找，from指定从哪个节点开始查找，它本身并不在查找行列中，只查找它后面的节点，如果设置为NULL表示从根节点开始查找。 */
struct device_node *of_find_node_by_name(struct device_node *from,
	const char *name)

/* 根据节点类型寻找 */
struct device_node *of_find_node_by_type(struct device_node *from,
	const char *type)

/* 根据节点类型和compatible属性寻找 */
struct device_node *of_find_compatible_node(struct device_node *from,
						const char *type, const char *compat)
```

```c
/* 寻找父节点 */
struct device_node *of_get_parent(const struct device_node *node)

/* 寻找子节点，寻找的是prev节点之后的节点。这是一个迭代寻找过程，例如寻找第二个子节点，这里就要填第一个子节点。参数为NULL 表示寻找第一个子节点。 */
struct device_node *of_get_next_child(const struct device_node *node,
					     struct device_node *prev);
```

### property
```c
struct property {
	char	*name;  属性名
	int	length;  属性长度
	void	*value;  属性值
	struct property *next;  下一个属性
#if defined(CONFIG_OF_DYNAMIC) || defined(CONFIG_SPARC)
	unsigned long _flags;
#endif
#if defined(CONFIG_OF_PROMTREE)
	unsigned int unique_id;
#endif
#if defined(CONFIG_OF_KOBJ)
	struct bin_attribute attr;
#endif
};
```



### 提取属性值

```c
#include <linux/of.h>

/* 获取节点属性 */
struct property *of_find_property(const struct device_node *np,
					 const char *name, int *lenp);


int of_property_read_u32(const struct device_node *np,
				       const char *propname, u32 *out_value)

int of_property_read_u8(const struct device_node *np, 
                        const char *propname, u8 *out_value)

int of_property_read_u16(const struct device_node *np, 
                         const char *propname, u16 *out_value)

/* 检查节点中某个属性是否存在 */
bool of_property_read_bool(const struct device_node *np,
					 const char *propname)

int of_property_read_string(const struct device_node *np, 
                 const char *propname, const char **out_string)

int of_property_read_u32_array(const struct device_node *np,
					     const char *propname,
					     u32 *out_values, size_t sz)
```



### gpio的获取

```c
#include <linux/of_gpio.h>

/* 根据gpio属性的名字获取 */
int of_get_named_gpio(struct device_node *np,
                             const char *propname, int index)

int of_get_gpio(struct device_node *np, int index)
```



### 地址的获取

在设备树的设备节点中大多会包含一些内存相关的属性，比如常用的reg属性。通常情况下，得到寄存器地址之后我们还要通过ioremap函数将物理地址转化为虚拟地址。现在内核提供了of函数，自动完成物理地址到虚拟地址的转换。

```c
struct resource {
    resource_size_t start;
    resource_size_t end;
    const char *name;
    unsigned long flags;
    unsigned long desc;
    struct resource *parent, *sibling, *child;
};


int of_address_to_resource(struct device_node *dev, int index,
			   struct resource *r)
```



### 中断的获取

```c
#include <linux/of_irq.h>


```



## DTC

DTC（Device Tree Compiler）为编译工具，它可以将.dts文件编译成.dtb文件。DTC的源码位于内核的scripts/dtc目录，内核选中CONFIG_OF，编译内核的时候，主机可执行程序DTC就会被编译出来。 即scripts/dtc/Makefile中。

```
hostprogs-y := dtc

always := $(hostprogs-y) 
```

在内核的arch/arm/boot/dts/Makefile中，若选中某种SOC，则与其对应相关的所有dtb文件都将编译出来。`make dtbs`可单独编译dtb。

```
ifeq ($(CONFIG_OF),y)

dtb-$(CONFIG_ARCH_EXYNOS4) += \
	exynos4412-origen.dtb \
	exynos4412-smdk4412.dtb \
	exynos4412-tiny4412.dtb \

endif
```

DTC在编译时，会对node进行合并操作，最终生成的dtb只有一个root node。



## DTB

dtb文件由头（Header）、结构块（device-tree structure）、字符串块（string block）三部分组成。

![dtb文件结构](.img/device-tree/dtb_struct.png)

### fdt_header

设备树文件头信息，存储在dtb文件的开头部分，使用fdt_header结构体描述。

```c
#define FDT_MAGIC   0xd00dfeed  /* 4: version, 4: total size */

struct fdt_header {
        fdt32_t magic;                   /* magic word FDT_MAGIC */
        fdt32_t totalsize;               /* total size of DT block */
        fdt32_t off_dt_struct;           /* offset to structure */
        fdt32_t off_dt_strings;          /* offset to strings */
        fdt32_t off_mem_rsvmap;          /* offset to memory reserve map */
        fdt32_t version;                 /* format version */
        fdt32_t last_comp_version;       /* last compatible version */

        /* version 2 fields below */
        fdt32_t boot_cpuid_phys;         /* Which physical CPU id we're
                                            booting on */
        /* version 3 fields below */
        fdt32_t size_dt_strings;         /* size of the strings block */

        /* version 17 fields below */
        fdt32_t size_dt_struct;          /* size of the structure block */
};
```

所有成员都是32位整型，并以大端模式存储；从头部可获知memory reservation block、structure block和strings block部分的起始地址和大小。

使用`fdtdump`工具对dtb文件进行分析，通过查看fdtdump输出信息以及dtb二进制文件信息。

 

![fdt_header和文件结构之间的关系](.img/device-tree/dtb_dump.png)

dtb文件结构分为header、fill_area、dt_struct及dt_string四个区域。header为头信息，其中fill_area为填充区域，填充数字0，dt_struct存储节点数值及名称相关信息，dt_string存储属性名。

### memory reservation block

预留内存区域，用于存放并保护一些重要的数据，与特定平台的实现相关。

```c
struct fdt_reserve_entry {
        fdt64_t address;
        fdt64_t size;
};
```

### structure block

structure block区域描述了设备树本身的结构和内容。

```c
struct fdt_node_header  {
    fdt32_t  tag; 
    /* node名称，以'\0'结尾的字符串形式，32-bits对齐，不够用0x0补齐 */
    char  name[0];
};
```

fdt_node_header结构体描述节点信息，tag是标识node的起始结束等信息的标志位，name指向node名称的首地址。tag的取值如下：

```c
#define FDT_BEGIN_NODE   0x1    /*  标识一个node的开始 */
#define FDT_END_NODE  0x2    /*  标识一个node的结束 */
#define FDT_PROP   0x3    /* 标识node节点下面的属性起始符 */
#define FDT_NOP    0x4    /* 所有的设备树解析程序都会忽略该令牌，一般用于覆盖树中的属性或者节点，以将其从树中删除 */
#define FDT_END    0x9  /* 标识structure block区域的结 */束
```

对于每个node节点的tag标识符一般为FDT_BEGIN_NODE，对于每个node节点下面的属性的tag标识符一般是FDT_PROP。

### strings block

strings block包含了设备树中所有属性的名称。

```c
struct  fdt_property  {
    fdt32_t  tag;
    /* 表示property value的长度 */
    fdt32_t  len;
    /* property的名称在string block的偏移 */
    fdt32_t  nameoff;
    /* property值，以'\0'结尾的字符串形式，32-bits对齐，用0x0补齐 */
    char  data[0];
};
```

属性用fdt_property结构体描述，tag标识是属性，取值为FDT_PROP；len为属性值的长度（包括‘\0’，单位字节）；nameoff为属性名称存储位置相对于off_dt_strings的偏移地址。

由于设备树node节点的名称和property value值的长度是不固定的，所以fdt_node_header和fdt_property结构体都使用零长数组的方式进行实现动态长度。

![dt_struct结构图](.img/device-tree/dtb_block.png)

### kernel解析设备树

bootloader在引导内核时，会预先读取dtb文件到内存，然后将地址通过bootm或bootz命令传给内核，`bootm [kernel_addr] [initrd_address] [dtb_address]`。

kernel会根据Device Tree的结构解析出能够使用的 property结构体。并根据Device Tree中的属性解析出数据来填充 property结构体，并将同一个node节点下面的所有属性通过property.next指针进行链接，形成一个单链表。

```c
struct property  {
    char  *name;                          /* property full name */
    int  length;                          /* property  value length */
    void  *value;                         /* property  value */
    struct  property *next;   /* next property  under the same node */
    unsigned  long _flags;
    unsigned  int unique_id;
    struct  bin_attribute attr;        /*属性文件，与sysfs文件系统挂接 */
};
```
kernel启动的入口函数为stext，其中会将bootloader传来的dtb地址（物理地址，保存在x21寄存器中）赋值给__fdt_pointer变量。此后执行的start_kernell()函数对dtb文件进行物理地址映射，并进行数据解析。

start_kernel()函数，在early_init_dt_scan_nodes()中会做以下三件事：

1. 扫描/chosen或者/chose@0节点下面的bootargs属性值到boot_command_line，此外，还处理initrd相关的property，并保存在initrd_start和initrd_end这两个全局变量中；

2. 扫描根节点下面，获取{size,address}-cells信息，并保存在dt_root_size_cells和dt_root_addr_cells全局变量中；

3. 扫描具有device_type = “memory”属性的/memory或者/memory@0节点下面的reg属性值，并把相关信息保存在meminfo中，全局变量meminfo保存了系统内存相关的信息。

Device Tree中的每一个node节点经过kernel处理都会生成一个device_node的结构体，device_node最终会被挂接到具体的device结构体。

```c
struct device_node  {
    const  char *name;              /* node的名称，取最后一次“/”和“@”之间子串 */
    const  char *type;              /* device_type的属性名称，没有为<NULL> */
    phandle  phandle;               /* phandle属性值 */
    const  char *full_name;   /*指向该结构体结束的位置，存放node的路径全名，例如：/chosen */
    struct  fwnode_handle fwnode;
 
    struct property *properties;  /*指向该节点下的第一个属性，其他属性与该属性链表相接 */
    struct property *deadprops;   /* removed properties */
    struct device_node *parent;   /*父节点 */
    struct device_node *child;    /*子节点 */
    struct device_node *sibling;  /*姊妹节点，与自己同等级的node */
    struct kobject kobj;          /* sysfs文件系统目录体现 */

    unsigned  long _flags;        /*当前node状态标志位 */
    void   *data;
};

#define OF_DYNAMIC        1 /* node and properties were  allocated via kmalloc */
#define OF_DETACHED       2  /* node has been detached from the device tree*/
#define OF_POPULATED      3  /* device already created for the node */
#define OF_POPULATED_BUS  4 /* of_platform_populate recursed to children of this node */
```



https://zhuanlan.zhihu.com/p/141623370

https://www.zhihu.com/column/c_1221056483527483392

http://www.wowotech.net/device_model/dt-code-analysis.html