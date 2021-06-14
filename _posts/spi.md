<!--
 * @Description: SPI
 * @Author: luo_u
 * @Date: 2020-08-07 15:51:29
 * @LastEditTime: 2020-10-06 20:51:59
-->


[TOC]

## spi框架


```
├── atmel-quadspi.c
├── internals.h
├── spi-altera.c
├── spi-axi-spi-engine.c
├── spi-bcm-qspi.c
├── spi-bcm-qspi.h
├── spi-bitbang.c
├── spi-bitbang-txrx.h
├── spi-brcmstb-qspi.c
├── spi-butterfly.c
├── spi.c
├── spi-cadence.c
├── spidev.c
├── spi-dln2.c
├── spi-dw.c
├── spi-dw.h
├── spi-dw-mid.c
├── spi-dw-mmio.c
├── spi-dw-pci.c
├── spi-fsl-cpm.c
├── spi-fsl-cpm.h
├── spi-fsl-dspi.c
├── spi-fsl-espi.c
├── spi-fsl-lib.c
├── spi-fsl-lib.h
├── spi-fsl-lpspi.c
├── spi-fsl-spi.c
├── spi-fsl-spi.h
├── spi-geni-qcom.c
├── spi-gpio.c
├── spi-img-spfi.c
├── spi-imx.c
├── spi-iproc-qspi.c
├── spi-jcore.c
├── spi-lantiq-ssc.c
├── spi-loopback-test.c
├── spi-mem.c
├── spi-meson-spicc.c
├── spi-meson-spifc.c
├── spi-mxic.c
├── spi-mxs.c
├── spi-npcm-pspi.c
├── spi-nuc900.c
├── spi-oc-tiny.c
├── spi-omap-100k.c
├── spi-omap2-mcspi.c
├── spi-omap-uwire.c
├── spi-orion.c
├── spi-qcom-qspi.c
├── spi-qup.c
├── spi-rb4xx.c
├── spi-rockchip.c
├── spi-rspi.c
├── spi-s3c24xx.c
├── spi-stm32.c
├── spi-xilinx.c
```

linux SPI驱动框架主要分为核心层，控制器驱动层以及设备驱动层。

![SPI驱动框架](.img/spi/spi-arch.jpg)

![SPI驱动框架](.img/spi/spi-arch2.jpg)



## spi核心层

SPI核心层代码位于 drivers/spi/spi.c，注册spi总线和spi_master设备类，同时提供spi设备驱动对spi总线操作的API。


### spi_alloc_master()
```c
static inline struct spi_controller *spi_alloc_master(struct device *host, unsigned int size)
{ 
   struct spi_controller *ctlr;
 
   ctlr = kzalloc(size + sizeof(*ctlr), GFP_KERNEL);
   if (!ctlr)
     return NULL;
 
   device_initialize(&ctlr->dev);
   ctlr->bus_num = -1;
   ctlr->num_chipselect = 1;
   ctlr->slave = slave;

   if (IS_ENABLED(CONFIG_SPI_SLAVE) && slave)
     ctlr->dev.class = &spi_slave_class;
   else 
     ctlr->dev.class = &spi_master_class;

   ctlr->dev.parent = dev;
   pm_suspend_ignore_children(&ctlr->dev, true);
   spi_controller_set_devdata(ctlr, &ctlr[1]);
 
   return ctlr;
} 
```



### spi_register_master()
```c
#define spi_register_master(_ctlr)	spi_register_controller(_ctlr)

int spi_register_controller(struct spi_controller *ctlr)
{
	struct device		*dev = ctlr->dev.parent;
	struct boardinfo	*bi;
	int			status = -ENODEV;
	int			id, first_dynamic;

	if (ctlr->num_chipselect == 0)  //判断片选个数 
		return -EINVAL;
	if (ctlr->bus_num >= 0) {  //验证spi总线编号 
		mutex_lock(&board_lock);
		id = idr_alloc(&spi_master_idr, ctlr, ctlr->bus_num,
			ctlr->bus_num + 1, GFP_KERNEL);
		mutex_unlock(&board_lock);
		if (WARN(id < 0, "couldn't get idr"))
			return id == -ENOSPC ? -EBUSY : id;
		ctlr->bus_num = id;
	}

	INIT_LIST_HEAD(&ctlr->queue);
	spin_lock_init(&ctlr->queue_lock);
	spin_lock_init(&ctlr->bus_lock_spinlock);
	mutex_init(&ctlr->bus_lock_mutex);
	mutex_init(&ctlr->io_mutex);
	ctlr->bus_lock_flag = 0;
	init_completion(&ctlr->xfer_completion);
	if (!ctlr->max_dma_len)
		ctlr->max_dma_len = INT_MAX;

	dev_set_name(&ctlr->dev, "spi%u", ctlr->bus_num);   //设置spi主机设备名
	status = device_add(&ctlr->dev);  //添加spi主机设备 

	if (ctlr->transfer) {
		dev_info(dev, "controller is unqueued, this is deprecated\n");
	} else if (ctlr->transfer_one || ctlr->transfer_one_message) {
		status = spi_controller_initialize_queue(ctlr);
	}

	spin_lock_init(&ctlr->statistics.lock);

	mutex_lock(&board_lock);
	list_add_tail(&ctlr->list, &spi_controller_list);  //spi主机list链表添加进全局spi_master_list链表
	list_for_each_entry(bi, &board_list, list)  //遍历全局board_list查找bi结构体 
		spi_match_controller_to_boardinfo(ctlr, &bi->board_info);  //找到匹配的板级spi设备 
	mutex_unlock(&board_lock);

}

static void spi_match_controller_to_boardinfo(struct spi_controller *ctlr,
					      struct spi_board_info *bi)
{
	struct spi_device *dev;

	if (ctlr->bus_num != bi->bus_num)  //判断是否所属的spi总线
		return;

	dev = spi_new_device(ctlr, bi);  //添加新的spi设备
}
```
将控制器添加到 spi_controller_list 链表，然后遍历 board_list 链表，如果匹配到spi_board_info，则添加一个spi设备。



### spi_register_board_info()
```c
int spi_register_board_info(struct spi_board_info const *info, unsigned n)
{
	struct boardinfo *bi;

	bi = kcalloc(n, sizeof(*bi), GFP_KERNEL);

	for (i = 0; i < n; i++, bi++, info++) {
		struct spi_controller *ctlr;

		memcpy(&bi->board_info, info, sizeof(*info));  //设置bi的板级信息
		if (info->properties) {
			bi->board_info.properties =
					property_entries_dup(info->properties);
			if (IS_ERR(bi->board_info.properties))
				return PTR_ERR(bi->board_info.properties);
		}

		mutex_lock(&board_lock);
		list_add_tail(&bi->list, &board_list);   //添加到全局board_list链表
		list_for_each_entry(ctlr, &spi_controller_list, list)
			spi_match_controller_to_boardinfo(ctlr, &bi->board_info); //遍历spi主机链表
		mutex_unlock(&board_lock);
	}

	return 0;
}

static void spi_match_controller_to_boardinfo(struct spi_controller *ctlr, struct spi_board_info *bi)
{
  struct spi_device *dev;

  if (ctlr->bus_num != bi->bus_num)
    return;

  dev = spi_new_device(ctlr, bi);
}
```
添加到 board_list 链表，然后遍历 spi_controller_list 链表，如果匹配到控制器，则调用[spi_new_device()](#spi_new_device)函数添加一个spi设备。



###　spi_new_device()
```c
struct spi_device *spi_new_device(struct spi_controller *ctlr,　struct spi_board_info *chip)
{
  struct spi_device *proxy;

  proxy = spi_alloc_device(ctlr);
  if (!proxy)
    return NULL;

  WARN_ON(strlen(chip->modalias) >= sizeof(proxy->modalias));

  proxy->chip_select = chip->chip_select;
  proxy->max_speed_hz = chip->max_speed_hz;
  proxy->mode = chip->mode;
  proxy->irq = chip->irq;
  strlcpy(proxy->modalias, chip->modalias, sizeof(proxy->modalias));
  proxy->dev.platform_data = (void *) chip->platform_data;
  proxy->controller_data = chip->controller_data;
  proxy->controller_state = NULL;

  if (chip->properties) {
    status = device_add_properties(&proxy->dev, chip->properties);
    if (status) {
      dev_err(&ctlr->dev,
        "failed to add properties to '%s': %d\n",　chip->modalias, status);
      goto err_dev_put;
    }
  }

  status = spi_add_device(proxy);
}


struct spi_device *spi_alloc_device(struct spi_controller *ctlr)                                              
{       
  struct spi_device *spi;
      
  if (!spi_controller_get(ctlr))
    return NULL;

  spi = kzalloc(sizeof(*spi), GFP_KERNEL);
  if (!spi) {
    spi_controller_put(ctlr);
    return NULL;
  }

  spi->master = spi->controller = ctlr;
  spi->dev.parent = &ctlr->dev;
  spi->dev.bus = &spi_bus_type;
  spi->dev.release = spidev_release;
  spi->cs_gpio = -ENOENT;
  
  spin_lock_init(&spi->statistics.lock);

  device_initialize(&spi->dev);
  return spi;
}

int spi_add_device(struct spi_device *spi)
{
  static DEFINE_MUTEX(spi_add_lock);
  struct spi_controller *ctlr = spi->controller;
  struct device *dev = ctlr->dev.parent;
  int status;

  /* Chipselects are numbered 0..max; validate. */
  if (spi->chip_select >= ctlr->num_chipselect) {
    dev_err(dev, "cs%d >= max %d\n", spi->chip_select,
      ctlr->num_chipselect);
    return -EINVAL;
  }

  /* Set the bus ID string */
  spi_dev_set_name(spi);

  mutex_lock(&spi_add_lock);

  status = bus_for_each_dev(&spi_bus_type, NULL, spi, spi_dev_check);　　//查找总线上的spi设备

  if (ctlr->cs_gpios)
    spi->cs_gpio = ctlr->cs_gpios[spi->chip_select];

  status = spi_setup(spi);　　　//调用spi主机 setup方法 

  status = device_add(&spi->dev);　　//添加设备
}   

```



### spi_register_driver()
```c
int spi_register_driver(struct module *owner, struct spi_driver *sdrv)
{
	sdrv->driver.owner = owner;
	sdrv->driver.bus = &spi_bus_type;
	if (sdrv->probe)
		sdrv->driver.probe = spi_drv_probe;
	if (sdrv->remove)
		sdrv->driver.remove = spi_drv_remove;
	if (sdrv->shutdown)
		sdrv->driver.shutdown = spi_drv_shutdown;
	return driver_register(&sdrv->driver);
}
```



### transfer_one()
```c
static int s3c64xx_spi_transfer_one(struct spi_master *master,
				    struct spi_device *spi,
				    struct spi_transfer *xfer)
{
	struct s3c64xx_spi_driver_data *sdd = spi_master_get_devdata(master);
	const unsigned int fifo_len = (FIFO_LVL_MASK(sdd) >> 1) + 1;
	const void *tx_buf = NULL;
	void *rx_buf = NULL;
	int target_len = 0, origin_len = 0;
	int use_dma = 0;
	int status;
	u32 speed;
	u8 bpw;
	unsigned long flags;

	reinit_completion(&sdd->xfer_completion);

	/* Only BPW and Speed may change across transfers */
	bpw = xfer->bits_per_word;
	speed = xfer->speed_hz;

	if (bpw != sdd->cur_bpw || speed != sdd->cur_speed) {
		sdd->cur_bpw = bpw;
		sdd->cur_speed = speed;
		sdd->cur_mode = spi->mode;
		s3c64xx_spi_config(sdd);
	}

	if (!is_polling(sdd) && (xfer->len > fifo_len) &&
	    sdd->rx_dma.ch && sdd->tx_dma.ch) {
		use_dma = 1;

	} else if (is_polling(sdd) && xfer->len > fifo_len) {
		tx_buf = xfer->tx_buf;
		rx_buf = xfer->rx_buf;
		origin_len = xfer->len;

		target_len = xfer->len;
		if (xfer->len > fifo_len)
			xfer->len = fifo_len;
	}

	do {
		spin_lock_irqsave(&sdd->lock, flags);

		/* Pending only which is to be done */
		sdd->state &= ~RXBUSY;
		sdd->state &= ~TXBUSY;

		s3c64xx_enable_datapath(sdd, xfer, use_dma);

		/* Start the signals */
		s3c64xx_spi_set_cs(spi, true);

		spin_unlock_irqrestore(&sdd->lock, flags);

		if (use_dma)
			status = s3c64xx_wait_for_dma(sdd, xfer);
		else
			status = s3c64xx_wait_for_pio(sdd, xfer);

		if (status) {
			dev_err(&spi->dev,
				"I/O Error: rx-%d tx-%d res:rx-%c tx-%c len-%d\n",
				xfer->rx_buf ? 1 : 0, xfer->tx_buf ? 1 : 0,
				(sdd->state & RXBUSY) ? 'f' : 'p',
				(sdd->state & TXBUSY) ? 'f' : 'p',
				xfer->len);

			if (use_dma) {
				if (xfer->tx_buf && (sdd->state & TXBUSY))
					dmaengine_terminate_all(sdd->tx_dma.ch);
				if (xfer->rx_buf && (sdd->state & RXBUSY))
					dmaengine_terminate_all(sdd->rx_dma.ch);
			}
		} else {
			s3c64xx_flush_fifo(sdd);
		}
		if (target_len > 0) {
			target_len -= xfer->len;

			if (xfer->tx_buf)
				xfer->tx_buf += xfer->len;

			if (xfer->rx_buf)
				xfer->rx_buf += xfer->len;

			if (target_len > fifo_len)
				xfer->len = fifo_len;
			else
				xfer->len = target_len;
		}
	} while (target_len > 0);

	if (origin_len) {
		/* Restore original xfer buffers and length */
		xfer->tx_buf = tx_buf;
		xfer->rx_buf = rx_buf;
		xfer->len = origin_len;
	}

	return status;
}


static void s3c64xx_flush_fifo(struct s3c64xx_spi_driver_data *sdd)
{
	void __iomem *regs = sdd->regs;
	unsigned long loops;
	u32 val;

	writel(0, regs + S3C64XX_SPI_PACKET_CNT);

	val = readl(regs + S3C64XX_SPI_CH_CFG);
	val &= ~(S3C64XX_SPI_CH_RXCH_ON | S3C64XX_SPI_CH_TXCH_ON);
	writel(val, regs + S3C64XX_SPI_CH_CFG);

	val = readl(regs + S3C64XX_SPI_CH_CFG);
	val |= S3C64XX_SPI_CH_SW_RST;
	val &= ~S3C64XX_SPI_CH_HS_EN;
	writel(val, regs + S3C64XX_SPI_CH_CFG);

	/* Flush TxFIFO*/
	loops = msecs_to_loops(1);
	do {
		val = readl(regs + S3C64XX_SPI_STATUS);
	} while (TX_FIFO_LVL(val, sdd) && loops--);

	if (loops == 0)
		dev_warn(&sdd->pdev->dev, "Timed out flushing TX FIFO\n");

	/* Flush RxFIFO*/
	loops = msecs_to_loops(1);
	do {
		val = readl(regs + S3C64XX_SPI_STATUS);
		if (RX_FIFO_LVL(val, sdd))
			readl(regs + S3C64XX_SPI_RX_DATA);
		else
			break;
	} while (loops--);

	if (loops == 0)
		dev_warn(&sdd->pdev->dev, "Timed out flushing RX FIFO\n");

	val = readl(regs + S3C64XX_SPI_CH_CFG);
	val &= ~S3C64XX_SPI_CH_SW_RST;
	writel(val, regs + S3C64XX_SPI_CH_CFG);

	val = readl(regs + S3C64XX_SPI_MODE_CFG);
	val &= ~(S3C64XX_SPI_MODE_TXDMA_ON | S3C64XX_SPI_MODE_RXDMA_ON);
	writel(val, regs + S3C64XX_SPI_MODE_CFG);
}
```



### spi_sync()
```c
static int spi_sync(struct spi_device *spi, struct spi_message *message)
{
	DECLARE_COMPLETION_ONSTACK(done);
	int status;
	struct spi_controller *ctlr = spi->controller;
	unsigned long flags;

	message->complete = spi_complete;
	message->context = &done;
	message->spi = spi;

	SPI_STATISTICS_INCREMENT_FIELD(&ctlr->statistics, spi_sync);
	SPI_STATISTICS_INCREMENT_FIELD(&spi->statistics, spi_sync);

	if (ctlr->transfer == spi_queued_transfer) {
		spin_lock_irqsave(&ctlr->bus_lock_spinlock, flags);

		trace_spi_message_submit(message);

		status = __spi_queued_transfer(spi, message, false);

		spin_unlock_irqrestore(&ctlr->bus_lock_spinlock, flags);
	} else {
		status = spi_async_locked(spi, message);
	}

	if (status == 0) {
		if (ctlr->transfer == spi_queued_transfer) {
			SPI_STATISTICS_INCREMENT_FIELD(&ctlr->statistics,
						       spi_sync_immediate);
			SPI_STATISTICS_INCREMENT_FIELD(&spi->statistics,
						       spi_sync_immediate);
			__spi_pump_messages(ctlr, false);
		}

		wait_for_completion(&done);
		status = message->status;
	}
	message->context = NULL;
	return status;
}


int spi_async_locked(struct spi_device *spi, struct spi_message *message)
{
	struct spi_controller *ctlr = spi->controller;

	spin_lock_irqsave(&ctlr->bus_lock_spinlock, flags);

	ret = __spi_async(spi, message);

	spin_unlock_irqrestore(&ctlr->bus_lock_spinlock, flags);

	return ret;
}

static int __spi_async(struct spi_device *spi, struct spi_message *message)
{
	struct spi_controller *ctlr = spi->controller;

	if (!ctlr->transfer)
		return -ENOTSUPP;

	message->spi = spi;

	SPI_STATISTICS_INCREMENT_FIELD(&ctlr->statistics, spi_async);
	SPI_STATISTICS_INCREMENT_FIELD(&spi->statistics, spi_async);

	trace_spi_message_submit(message);

	return ctlr->transfer(spi, message);
}
```
spi_sync() 是同步的，会等待传输完成，而 spi_async() 会立即返回，不等待发送完成。spi_sync() 就是在 spi_async() 基础上加了自旋锁，最终调用的还是控制器的 transfer() 函数。




## spi总线


### 注册SPI总线
```c
struct bus_type spi_bus_type = {
	.name		= "spi",
	.dev_groups	= spi_dev_groups,
	.match		= spi_match_device,
	.uevent		= spi_uevent,
};

static struct class spi_master_class = {
	.name		= "spi_master",
	.owner		= THIS_MODULE,
	.dev_release	= spi_controller_release,
	.dev_groups	= spi_master_groups,
};

#ifdef CONFIG_SPI_SLAVE
static struct class spi_slave_class = {
	.name		= "spi_slave",
	.owner		= THIS_MODULE,
	.dev_release	= spi_controller_release,
	.dev_groups	= spi_slave_groups,
};
#else
extern struct class spi_slave_class;	/* dummy */
#endif

static int __init spi_init(void)  
{
	buf = kmalloc(SPI_BUFSIZ, GFP_KERNEL);  //分配数据收发缓冲区

	status = bus_register(&spi_bus_type);  //注册spi总线 

	status = class_register(&spi_master_class);  //注册spi主机类

	if (IS_ENABLED(CONFIG_SPI_SLAVE)) {
		status = class_register(&spi_slave_class);
	}

	return 0;
}
```
注册SPI总线，成功注册后，在/sys/bus下即可找到spi目录。注册设备类，成功注册后，在/sys/class目录下即可找到spi_master目录。

```c
static int spi_match_device(struct device *dev, struct device_driver *drv)
{
	const struct spi_device	*spi = to_spi_device(dev);
	const struct spi_driver	*sdrv = to_spi_driver(drv);

	/* Check override first, and if set, only use the named driver */
	if (spi->driver_override)
		return strcmp(spi->driver_override, drv->name) == 0;

	if (of_driver_match_device(dev, drv))
		return 1;

	if (acpi_driver_match_device(dev, drv))
		return 1;

	if (sdrv->id_table)
		return !!spi_match_id(sdrv->id_table, spi);

	return strcmp(spi->modalias, drv->name) == 0;
}
```
总线匹配规则：

1. 比较驱动中的of_match_table的compatible和device的of_node的compatible。
2. 比较驱动中的acpi_match_table的compatible和device的of_node的compatible。
3. 判断驱动中是否支持id数组，如果支持，查找匹配此id的spi_device。
4. 比较设备的modalias的和驱动的名字是否相同。



## spi主机

### spi_master

```c
struct spi_master { 
    struct device   dev;    //spi主机设备文件 
    struct list_head list; 
    s16 bus_num;    //spi总线号 
    u16 num_chipselect; //片选号 
    u16 dma_alignment;  //dma算法 
    u16 mode_bits;  //模式位 
    u16 flags;  //传输类型标志 
    spinlock_t  bus_lock_spinlock;  //spi总线自旋锁 
    struct mutex    bus_lock_mutex; //spi总线互斥锁 
    bool    bus_lock_flag;  //上锁标志 
    int (*setup)(struct spi_device *spi);   //setup方法 
    int (*transfer)(struct spi_device *spi,struct spi_message *mesg);   //传输方法 
    void    (*cleanup)(struct spi_device *spi); //cleanup方法 
}; 

```

### spi_master注册
```c
static int s3c64xx_spi_probe(struct platform_device *pdev)
{
	struct spi_master *master;

	mem_res = platform_get_resource(pdev, IORESOURCE_MEM, 0);

	irq = platform_get_irq(pdev, 0);

	master = spi_alloc_master(&pdev->dev,spi_register_master()
				sizeof(struct s3c64xx_spi_driver_data));

	platform_set_drvdata(pdev, master);

	master->dev.of_node = pdev->dev.of_node;
	master->bus_num = sdd->port_id;
	master->setup = s3c64xx_spi_setup;
	master->cleanup = s3c64xx_spi_cleanup;
	master->prepare_transfer_hardware = s3c64xx_spi_prepare_transfer;
	master->prepare_message = s3c64xx_spi_prepare_message;
	master->transfer_one = s3c64xx_spi_transfer_one;
	master->num_chipselect = sci->num_cs;
	master->dma_alignment = 8;
	master->bits_per_word_mask = SPI_BPW_MASK(32) | SPI_BPW_MASK(16) |
					SPI_BPW_MASK(8);

	master->mode_bits = SPI_CPOL | SPI_CPHA | SPI_CS_HIGH;
	master->auto_runtime_pm = true;
	if (!is_polling(sdd))
		master->can_dma = s3c64xx_spi_can_dma;

	ret = spi_register_controller(master);

	return 0;
}


static struct platform_driver s3c64xx_spi_driver = {
	.driver = {
		.name	= "s3c64xx-spi",
		.pm = &s3c64xx_spi_pm,
		.of_match_table = of_match_ptr(s3c64xx_spi_dt_match),
	},
	.probe = s3c64xx_spi_probe,
	.remove = s3c64xx_spi_remove,
	.id_table = s3c64xx_spi_driver_ids,
};

module_platform_driver(s3c64xx_spi_driver);
```
使用[spi_alloc_master()](#spi_alloc_master)函数实例化一个spi控制器，初始化中断和时钟，实现[transfer_one()](#transfer_one)函数，最后会使用[spi_register_master()](#spi_register_master)函数注册。注册成功后/sys/class/spi_master下会出现spi控制器。



## spi设备

### spi_device
```c
struct spi_device {
    struct device   dev;    //设备文件 
    struct spi_master   *master;    //spi主机 
    u32 max_speed_hz;   //最大速率 
    u8  chip_select;    //片选 
    u8  mode;   //模式 
#define	SPI_CPHA	0x01			/* clock phase */
#define	SPI_CPOL	0x02			/* clock polarity */
#define	SPI_MODE_0	(0|0)			/* (original MicroWire) */
#define	SPI_MODE_1	(0|SPI_CPHA)
#define	SPI_MODE_2	(SPI_CPOL|0)
#define	SPI_MODE_3	(SPI_CPOL|SPI_CPHA)
#define	SPI_CS_HIGH	0x04			/* chipselect active high? */
#define	SPI_LSB_FIRST	0x08			/* per-word bits-on-wire */
#define	SPI_3WIRE	0x10			/* SI/SO signals shared */
#define	SPI_LOOP	0x20			/* loopback mode */
#define	SPI_NO_CS	0x40			/* 1 dev/bus, no chipselect */
#define	SPI_READY	0x80			/* slave pulls low to pause */
#define	SPI_TX_DUAL	0x100			/* transmit with 2 wires */
#define	SPI_TX_QUAD	0x200			/* transmit with 4 wires */
#define	SPI_RX_DUAL	0x400			/* receive with 2 wires */
#define	SPI_RX_QUAD	0x800			/* receive with 4 wires */
#define	SPI_CS_WORD	0x1000			/* toggle cs after each word */
#define	SPI_TX_OCTAL	0x2000			/* transmit with 8 wires */
#define	SPI_RX_OCTAL	0x4000			/* receive with 8 wires */
#define	SPI_3WIRE_HIZ	0x8000			/* high impedance turnaround */
    u8  bits_per_word;  //一个字有多少位 
    int irq;    //中断号 
    void    *controller_state;  //控制器状态 
    void    *controller_data;   //控制器数据 
    char    modalias[SPI_NAME_SIZE]; //名字 
};
```




### spi_board_info

```c
struct spi_board_info { 
    char    modalias[SPI_NAME_SIZE];    //名字 
    const void  *platform_data; //平台数据 
    void    *controller_data;   //控制器数据 
    int irq;            //中断号 
    u32 max_speed_hz;   //最大速率 
    u16 bus_num;        //spi总线编号 
    u16 chip_select;    //片选 
    u8  mode;           //模式 
}; 
```



### spi_device注册

spi_device的注册一般在板文件中实现，先填充 spi_board_info 结构体，再使用[spi_register_board_info()](#spi_register_board_info)函数添加。

```c
static struct spi_board_info spi_board_info[] __initdata = {
	{
		.modalias	= "lms501kf03",
		.platform_data	= NULL,
		.max_speed_hz	= 1200000,
		.bus_num	= LCD_BUS_NUM,
		.chip_select	= 0,
		.mode		= SPI_MODE_3,
		.controller_data = (void *)DISPLAY_CS,
	}
};

spi_register_board_info(spi_board_info, ARRAY_SIZE(spi_board_info));
```



## spi设备驱动

### spi_driver

```c
struct spi_driver {
	const struct spi_device_id *id_table;
	int			(*probe)(struct spi_device *spi);
	int			(*remove)(struct spi_device *spi);
	void			(*shutdown)(struct spi_device *spi);
	struct device_driver	driver;
};
```

### spi_driver 注册

spi设备驱动根据功能分开在各个地方，如spi flash的一般就在/drivers/mtd/device/下，siwtch的spi接口就放在/drivers/net/ethernet/下。

```c
static struct spi_driver ak_spiflash_driver = {
	.driver = {
		.name	= "ak-spiflash",
		.bus	= &spi_bus_type,
		.owner	= THIS_MODULE,
	},
	.probe	= ak_spiflash_probe,
	.remove	= __devexit_p(ak_spiflash_remove),
};


static int __init ak_spiflash_init(void)
{
	return spi_register_driver(&ak_spiflash_driver);
}
```



## spi消息传输

```c
struct spi_message { 
    struct list_head    transfers;  //spi传输事务链表头 
    struct spi_device   *spi;   //所属spi设备 
    unsigned    is_dma_mapped:1; 
    void    (*complete)(void *context);  
    void    *context; 
    unsigned    actual_length; 
    int     status; //传输状态 
    struct list_head    queue; 
    void    *state; 
}; 
```

### spi_transfer
```c
struct spi_transfer { 
    const void  *tx_buf;    //发送缓冲区指针 
    void        *rx_buf;    //接收缓冲区指针 
    unsigned    len;    //消息长度 
    dma_addr_t  tx_dma; //DMA发送地址 
    dma_addr_t  rx_dma; //DMA接收地址 
    unsigned    cs_change:1;     
    u8      bits_per_word;  //一个字多少位 
    u16     delay_usecs;    //毫秒级延时 
    u32     speed_hz;   //速率 
    struct list_head transfer_list; //传输链表头 
}; 
```



## spidev.c

注册SPI设备驱动代码位于/drivers/spi/spidev.c，同时创建spi字符设备文件，为应用层提供文件操作接口。

```c
static const struct file_operations spidev_fops = {
	.owner =	THIS_MODULE,
	.write =	spidev_write,
	.read =		spidev_read,
	.unlocked_ioctl = spidev_ioctl,
	.compat_ioctl = spidev_compat_ioctl,
	.open =		spidev_open,
	.release =	spidev_release,
	.llseek =	no_llseek,
};

static int spidev_probe(struct spi_device *spi)
{
	struct spidev_data	*spidev;
	int			status;
	unsigned long		minor;

	spidev = kzalloc(sizeof(*spidev), GFP_KERNEL);

	spidev->spi = spi;
	spin_lock_init(&spidev->spi_lock);
	mutex_init(&spidev->buf_lock);
	INIT_LIST_HEAD(&spidev->device_entry);

	mutex_lock(&device_list_lock);
	minor = find_first_zero_bit(minors, N_SPI_MINORS);
	if (minor < N_SPI_MINORS) {
		struct device *dev;

		spidev->devt = MKDEV(SPIDEV_MAJOR, minor);
		dev = device_create(spidev_class, &spi->dev, spidev->devt,
				    spidev, "spidev%d.%d",
				    spi->master->bus_num, spi->chip_select);
		status = PTR_ERR_OR_ZERO(dev);
	} else {
		dev_dbg(&spi->dev, "no minor number available!\n");
		status = -ENODEV;
	}
	if (status == 0) {
		set_bit(minor, minors);
		list_add(&spidev->device_entry, &device_list);
	}
	mutex_unlock(&device_list_lock);

	spidev->speed_hz = spi->max_speed_hz;

	if (status == 0)
		spi_set_drvdata(spi, spidev);
	else
		kfree(spidev);

	return status;
}

static struct spi_driver spidev_spi_driver = {
	.driver = {
		.name =		"spidev",
		.of_match_table = of_match_ptr(spidev_dt_ids),
		.acpi_match_table = ACPI_PTR(spidev_acpi_ids),
	},
	.probe =	spidev_probe,
	.remove =	spidev_remove,
};

static int __init spidev_init(void)
{
	BUILD_BUG_ON(N_SPI_MINORS > 256);
	status = register_chrdev(SPIDEV_MAJOR, "spi", &spidev_fops);

	spidev_class = class_create(THIS_MODULE, "spidev");

	status = spi_register_driver(&spidev_spi_driver);

	return status;
}
module_init(spidev_init);
```


### read()

```c
static inline ssize_t spidev_sync_read(struct spidev_data *spidev, size_t len)
{
	struct spi_transfer	t = {
			.rx_buf		= spidev->rx_buffer,
			.len		= len,
			.speed_hz	= spidev->speed_hz,
		};
	struct spi_message	m;

	spi_message_init(&m);
	spi_message_add_tail(&t, &m);
	return spidev_sync(spidev, &m);
}

static ssize_t spidev_read(struct file *filp, char __user *buf, size_t count, loff_t *f_pos)
{
	struct spidev_data	*spidev;
	ssize_t			status = 0;

	/* chipselect only toggles at start or end of operation */
	if (count > bufsiz)
		return -EMSGSIZE;

	spidev = filp->private_data;

	mutex_lock(&spidev->buf_lock);
	status = spidev_sync_read(spidev, count);
	if (status > 0) {
		unsigned long	missing;

		missing = copy_to_user(buf, spidev->rx_buffer, status);
		if (missing == status)
			status = -EFAULT;
		else
			status = status - missing;
	}
	mutex_unlock(&spidev->buf_lock);

	return status;
}

static ssize_t spidev_sync(struct spidev_data *spidev, struct spi_message *message)
{
	int status;
	struct spi_device *spi;

	spin_lock_irq(&spidev->spi_lock);
	spi = spidev->spi;
	spin_unlock_irq(&spidev->spi_lock);

	if (spi == NULL)
		status = -ESHUTDOWN;
	else
		status = spi_sync(spi, message);

	if (status == 0)
		status = message->actual_length;

	return status;
}
```


### write()

```c
static ssize_t spidev_write(struct file *filp, const char __user *buf,
		size_t count, loff_t *f_pos)
{
	struct spidev_data	*spidev;
	ssize_t			status = 0;
	unsigned long		missing;

	/* chipselect only toggles at start or end of operation */
	if (count > bufsiz)
		return -EMSGSIZE;

	spidev = filp->private_data;

	mutex_lock(&spidev->buf_lock);
	missing = copy_from_user(spidev->tx_buffer, buf, count);
	if (missing == 0)
		status = spidev_sync_write(spidev, count);
	else
		status = -EFAULT;
	mutex_unlock(&spidev->buf_lock);

	return status;
}

static inline ssize_t spidev_sync_write(struct spidev_data *spidev, size_t len)
{
	struct spi_transfer	t = {
			.tx_buf		= spidev->tx_buffer,
			.len		= len,
			.speed_hz	= spidev->speed_hz,
		};
	struct spi_message	m;

	spi_message_init(&m);
	spi_message_add_tail(&t, &m);
	return spidev_sync(spidev, &m);
}
```




## 参考

https://blog.csdn.net/weixin_42262944/article/details/102944484

https://www.jianshu.com/p/60d2d5849db9

http://blog.chinaunix.net/uid-31418466-id-5786163.html