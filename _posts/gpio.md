[TOC]

## gpiolib

gpiolib的作用是对所有的gpio实行统一管理，因为驱动在工作的时候，会出现好几个驱动共同使用同一个gpio的情况，这会造成混乱，所以内核提供了一些方法来管理gpio资源。代码实现在gpio/gpiolib.c中。

### gpio_desc

```c
struct gpio_desc {
	struct gpio_chip	*chip;
	unsigned long		flags;
/* flag symbols are bit numbers */
#define FLAG_REQUESTED	0
#define FLAG_IS_OUT	1
#define FLAG_RESERVED	2
#define FLAG_EXPORT	3	/* protected by sysfs_lock */
#define FLAG_SYSFS	4	/* exported via /sys/class/gpio/control */
#define FLAG_TRIG_FALL	5	/* trigger on falling edge */
#define FLAG_TRIG_RISE	6	/* trigger on rising edge */
#define FLAG_ACTIVE_LOW	7	/* sysfs value has active low */

#define ID_SHIFT	16	/* add new flags before this one */

#define GPIO_FLAGS_MASK		((1 << ID_SHIFT) - 1)
#define GPIO_TRIGGER_MASK	(BIT(FLAG_TRIG_FALL) | BIT(FLAG_TRIG_RISE))

#ifdef CONFIG_DEBUG_FS
	const char		*label;
#endif
};

static struct gpio_desc gpio_desc[ARCH_NR_GPIOS];
```

定义了一个gpio_desc结构体数组。



### gpio_chip

gpio_chip 结构体包含了对gpio的各种操作方法，比如设置gpio的输入输出，设置gpio的输出值或者获取值等。

```c
struct gpio_chip {
	const char		*label;
	struct device		*dev;
	struct module		*owner;

	int			(*request)(struct gpio_chip *chip,
						unsigned offset);
	void			(*free)(struct gpio_chip *chip,
						unsigned offset);

	int			(*direction_input)(struct gpio_chip *chip,
						unsigned offset);
	int			(*get)(struct gpio_chip *chip,
						unsigned offset);
	int			(*direction_output)(struct gpio_chip *chip,
						unsigned offset, int value);
	int			(*set_debounce)(struct gpio_chip *chip,
						unsigned offset, unsigned debounce);

	void			(*set)(struct gpio_chip *chip,
						unsigned offset, int value);

	int			(*to_irq)(struct gpio_chip *chip,
						unsigned offset);

	void			(*dbg_show)(struct seq_file *s,
						struct gpio_chip *chip);
	int			base;
	u16			ngpio;
	const char		*const *names;
	unsigned		can_sleep:1;
	unsigned		exported:1;

#if defined(CONFIG_OF_GPIO)
	/*
	 * If CONFIG_OF is enabled, then all GPIO controllers described in the
	 * device tree automatically may have an OF translation
	 */
	struct device_node *of_node;
	int of_gpio_n_cells;
	int (*of_xlate)(struct gpio_chip *gc, struct device_node *np,
		        const void *gpio_spec, u32 *flags);
#endif
};
```



### gpiochip_add()
```c
int gpiochip_add(struct gpio_chip *chip)
{
	unsigned long	flags;
	int		status = 0;
	unsigned	id;
	int		base = chip->base;

	if ((!gpio_is_valid(base) || !gpio_is_valid(base + chip->ngpio - 1))
			&& base >= 0) {
		status = -EINVAL;
		goto fail;
	}

	spin_lock_irqsave(&gpio_lock, flags);

	if (base < 0) {
		base = gpiochip_find_base(chip->ngpio);
		if (base < 0) {
			status = base;
			goto unlock;
		}
		chip->base = base;
	}

	/* these GPIO numbers must not be managed by another gpio_chip */
	for (id = base; id < base + chip->ngpio; id++) {
		if (gpio_desc[id].chip != NULL) {
			status = -EBUSY;
			break;
		}
	}
	if (status == 0) {
		for (id = base; id < base + chip->ngpio; id++) {
			gpio_desc[id].chip = chip;

			/* REVISIT:  most hardware initializes GPIOs as
			 * inputs (often with pullups enabled) so power
			 * usage is minimized.  Linux code should set the
			 * gpio direction first thing; but until it does,
			 * we may expose the wrong direction in sysfs.
			 */
			gpio_desc[id].flags = !chip->direction_input
				? (1 << FLAG_IS_OUT)
				: 0;
		}
	}

	of_gpiochip_add(chip);

unlock:
	spin_unlock_irqrestore(&gpio_lock, flags);

	if (status)
		goto fail;

	status = gpiochip_export(chip);
	if (status)
		goto fail;

	return 0;
}
```

 

### gpio_request()
```c
int gpio_request(unsigned gpio, const char *label)
{
	struct gpio_desc	*desc;
	struct gpio_chip	*chip;
	int			status = -EINVAL;
	unsigned long		flags;

	spin_lock_irqsave(&gpio_lock, flags);

	if (!gpio_is_valid(gpio))
		goto done;
	desc = &gpio_desc[gpio];
	chip = desc->chip;
	if (chip == NULL)
		goto done;

	if (!try_module_get(chip->owner))
		goto done;

	/* NOTE:  gpio_request() can be called in early boot,
	 * before IRQs are enabled, for non-sleeping (SOC) GPIOs.
	 */

	if (test_and_set_bit(FLAG_REQUESTED, &desc->flags) == 0) {
		desc_set_label(desc, label ? : "?");
		status = 0;
	} else {
		status = -EBUSY;
		module_put(chip->owner);
		goto done;
	}

	if (chip->request) {
		/* chip->request may sleep */
		spin_unlock_irqrestore(&gpio_lock, flags);
		status = chip->request(chip, gpio - chip->base);
		spin_lock_irqsave(&gpio_lock, flags);

		if (status < 0) {
			desc_set_label(desc, NULL);
			module_put(chip->owner);
			clear_bit(FLAG_REQUESTED, &desc->flags);
		}
	}

done:
	if (status)
		pr_debug("gpio_request: gpio-%d (%s) status %d\n",
			gpio, label ? : "?", status);
	spin_unlock_irqrestore(&gpio_lock, flags);
	return status;
}
```



### gpio_free()
```c
void gpio_free(unsigned gpio)
{
	unsigned long		flags;
	struct gpio_desc	*desc;
	struct gpio_chip	*chip;

	might_sleep();

	if (!gpio_is_valid(gpio)) {
		WARN_ON(extra_checks);
		return;
	}

	gpio_unexport(gpio);

	spin_lock_irqsave(&gpio_lock, flags);

	desc = &gpio_desc[gpio];
	chip = desc->chip;
	if (chip && test_bit(FLAG_REQUESTED, &desc->flags)) {
		if (chip->free) {
			spin_unlock_irqrestore(&gpio_lock, flags);
			might_sleep_if(chip->can_sleep);
			chip->free(chip, gpio - chip->base);
			spin_lock_irqsave(&gpio_lock, flags);
		}
		desc_set_label(desc, NULL);
		module_put(desc->chip->owner);
		clear_bit(FLAG_ACTIVE_LOW, &desc->flags);
		clear_bit(FLAG_REQUESTED, &desc->flags);
	} else
		WARN_ON(extra_checks);

	spin_unlock_irqrestore(&gpio_lock, flags);
}
```



### gpio_direction_input()

```c
int gpio_direction_input(unsigned gpio)
{
	unsigned long		flags;
	struct gpio_chip	*chip;
	struct gpio_desc	*desc = &gpio_desc[gpio];
	int			status = -EINVAL;

	spin_lock_irqsave(&gpio_lock, flags);

	if (!gpio_is_valid(gpio))
		goto fail;
	chip = desc->chip;
	if (!chip || !chip->get || !chip->direction_input)
		goto fail;
	gpio -= chip->base;
	if (gpio >= chip->ngpio)
		goto fail;
	status = gpio_ensure_requested(desc, gpio);
	if (status < 0)
		goto fail;

	/* now we know the gpio is valid and chip won't vanish */

	spin_unlock_irqrestore(&gpio_lock, flags);

	might_sleep_if(chip->can_sleep);

	if (status) {
		status = chip->request(chip, gpio);
		if (status < 0) {
			pr_debug("GPIO-%d: chip request fail, %d\n",
				chip->base + gpio, status);
			/* and it's not available to anyone else ...
			 * gpio_request() is the fully clean solution.
			 */
			goto lose;
		}
	}

	status = chip->direction_input(chip, gpio);
	if (status == 0)
		clear_bit(FLAG_IS_OUT, &desc->flags);

	trace_gpio_direction(chip->base + gpio, 1, status);
lose:
	return status;
fail:
	spin_unlock_irqrestore(&gpio_lock, flags);
	if (status)
		pr_debug("%s: gpio-%d status %d\n",
			__func__, gpio, status);
	return status;
}
```



### gpio_direction_output()

```c
int gpio_direction_output(unsigned gpio, int value)
{
	unsigned long		flags;
	struct gpio_chip	*chip;
	struct gpio_desc	*desc = &gpio_desc[gpio];
	int			status = -EINVAL;

	spin_lock_irqsave(&gpio_lock, flags);

	if (!gpio_is_valid(gpio))
		goto fail;
	chip = desc->chip;
	if (!chip || !chip->set || !chip->direction_output)
		goto fail;
	gpio -= chip->base;
	if (gpio >= chip->ngpio)
		goto fail;
	status = gpio_ensure_requested(desc, gpio);
	if (status < 0)
		goto fail;

	/* now we know the gpio is valid and chip won't vanish */

	spin_unlock_irqrestore(&gpio_lock, flags);

	might_sleep_if(chip->can_sleep);

	if (status) {
		status = chip->request(chip, gpio);
		if (status < 0) {
			pr_debug("GPIO-%d: chip request fail, %d\n",
				chip->base + gpio, status);
			/* and it's not available to anyone else ...
			 * gpio_request() is the fully clean solution.
			 */
			goto lose;
		}
	}

	status = chip->direction_output(chip, gpio, value);
	if (status == 0)
		set_bit(FLAG_IS_OUT, &desc->flags);
	trace_gpio_value(chip->base + gpio, 0, value);
	trace_gpio_direction(chip->base + gpio, 0, status);
lose:
	return status;
fail:
	spin_unlock_irqrestore(&gpio_lock, flags);
	if (status)
		pr_debug("%s: gpio-%d status %d\n",
			__func__, gpio, status);
	return status;
}
```



### gpio_get_value()
```c
int gpio_get_value(unsigned gpio)
{
	struct gpio_chip	*chip;
	int value;

	chip = gpio_to_chip(gpio);
	WARN_ON(chip->can_sleep);
	value = chip->get ? chip->get(chip, gpio - chip->base) : 0;
	trace_gpio_value(gpio, 1, value);
	return value;
}
```



### gpio_set_value()
```c
void gpio_set_value(unsigned gpio, int value)
{
	struct gpio_chip	*chip;

	chip = gpio_to_chip(gpio);
	WARN_ON(chip->can_sleep);
	trace_gpio_value(gpio, 0, value);
	chip->set(chip, gpio - chip->base, value);
}
```





## gpio注册 

```c
static struct s3c_gpio_chip exynos4_gpio_common_4bit[] = {
#endif
	{
		.base	= S5P_VA_GPIO1,
		.eint_offset = 0x0,
		.group	= 0,
		.chip	= {
			.base	= EXYNOS4_GPA0(0),
			.ngpio	= EXYNOS4_GPIO_A0_NR,
			.label	= "GPA0",
		},
	}, {
		.base	= (S5P_VA_GPIO1 + 0x20),
		.eint_offset = 0x4,
		.group	= 1,
		.chip	= {
			.base	= EXYNOS4_GPA1(0),
			.ngpio	= EXYNOS4_GPIO_A1_NR,
			.label	= "GPA1",
		},
	},
    ...
};

static __init int exynos4_gpiolib_init(void)
{
	struct s3c_gpio_chip *chip;
	int i;
	int nr_chips;

	chip = exynos4_gpio_common_4bit;
	nr_chips = ARRAY_SIZE(exynos4_gpio_common_4bit);

	for (i = 0; i < nr_chips; i++, chip++) {
		if (chip->config == NULL)
			chip->config = &gpio_cfg;
		if (chip->base == NULL)
			pr_err("No allocation of base address for [common gpio]");
	}

	samsung_gpiolib_add_4bit_chips(exynos4_gpio_common_4bit, nr_chips);

	return 0;
}

core_initcall(exynos4_gpiolib_init);
```

```c
void __init samsung_gpiolib_add_4bit2_chips(struct s3c_gpio_chip *chip,
					    int nr_chips)
{
	for (; nr_chips > 0; nr_chips--, chip++) {
		samsung_gpiolib_add_4bit2(chip);
		s3c_gpiolib_add(chip);
	}
}

__init void s3c_gpiolib_add(struct s3c_gpio_chip *chip)
{
	struct gpio_chip *gc = &chip->chip;

	spin_lock_init(&chip->lock);

	if (!gc->direction_input)
		gc->direction_input = s3c_gpiolib_input;
	if (!gc->direction_output)
		gc->direction_output = s3c_gpiolib_output;
	if (!gc->set)
		gc->set = s3c_gpiolib_set;
	if (!gc->get)
		gc->get = s3c_gpiolib_get;

	gpiochip_add(gc);
}
```
填充gpio_chip结构体，再用[gpiochip_add()](#gpiochip_add)函数注册到内核。需要确定每组gpio的虚拟地址，其中物理地址和虚拟地址的映射平台已经做好了。

```c
#define S5P_VA_GPIO1		S3C_ADDR(0x02200000)
#define S5P_VA_GPIO2		S3C_ADDR(0x02240000)

static struct map_desc exynos4_iodesc[] __initdata = {
    {
        .virtual	= (unsigned long)S5P_VA_GPIO1,
        .pfn		= __phys_to_pfn(EXYNOS4_PA_GPIO1),
        .length		= SZ_4K,
        .type		= MT_DEVICE,
    }, {
        .virtual	= (unsigned long)S5P_VA_GPIO2,
        .pfn		= __phys_to_pfn(EXYNOS4_PA_GPIO2),
        .length		= SZ_4K,
        .type		= MT_DEVICE,
    }, 
};

void __init exynos4_map_io(void)
{
	iotable_init(exynos4_iodesc, ARRAY_SIZE(exynos4_iodesc));
}
```



## pinctrl

