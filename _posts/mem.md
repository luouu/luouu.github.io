<!--

 * @Description: 内存与I/O访问
 * @Author: luo_u
 * @Date: 2020-06-01 10:49:24
 * @LastEditTime: 2020-07-15 22:23:07
--> 

[TOC]

## 内存分配
```c
void *kmalloc(size_t size,　 gfp_t flags);
void kfree(const void *objp);

static inline void *kzalloc(size_t size, gfp_t flags)
{    
	return kmalloc(size, flags | __GFP_ZERO);
}
```
kmalloc在物理内存映射区域申请一个连续存储空间，保留内存原有的数据，不清零。分配的内存最小32Byte，最大128KB。flags参数：
- GFP_ATOMIC  原子性分配内存，分配内存的过程不会被高优先级进程或中断打断。若不存在空闲页，则不等待，直接返回。
- GFP_KERNEL  在内核空间的进程中申请内存，若不存在空闲页会引起阻塞，因此不能在中断上下文或持有自旋锁时使用。
- GFP_DMA  给DMA控制器分配内存（DMA要求分配虚拟地址和物理地址连续）。

kzalloc() 会对申请到的内存内容清零。

```c
void *vmalloc(unsigned long size);
void vfree(const void *addr);
```
vmalloc() 能在高端内存区分配，用来分配虚拟地址空间连续，而物理地址不一定连续的内存。常用于申请大内存空间。因为要建立新的页表，将不连续的物理内存映射成连续的虚拟内存，所以开销比较大。分配内存时则可能产生阻塞，因此不能从中断上下文调用。




## 虚拟地址与物理地址
```c
typedef unsigned long long phys_addr_t;

/*内核虚拟地址转化为物理地址*/
static inline phys_addr_t virt_to_phys(const volatile void *x)
{
	return __virt_to_phys((unsigned long)(x));
}

/*物理地址转化为内核虚拟地址*/
static inline void *phys_to_virt(phys_addr_t x)
{
	return (void *)__phys_to_virt(x);
}
```





## I/O端口

设备通常会提供一组寄存器来用于控制设备、读写设备和获取设备状态，即控制寄存器、数据寄存器和状态寄存器。这些寄存器可能位于 I/O 空间，也可能位于内存空间。当位于 I/O 空间时，通常被称为 I/O 端口，端口号标识了外设的寄存器地址。位于内存空间时，对应的内存空间被称为 I/O 内存。内存空间是必须的，而 I/O 空间是可选的。

I/O 端口访问的一种途径是直接使用 I/O 端口操作函数，在设备打开或驱动模块被加载时申请
I/O 端口区域，之后使用 inb()、outb()等进行端口访问，最后在设备关闭或驱动被卸载时释放 I/O
端口范围。

<img src=".img/mem/io_port.png" style="zoom:50%;" />

```c
struct resource *request_region(unsigned long start, unsigned long n, const char *name);

void release_region(unsigned long start, unsigned long n);
```
request_region()函数向内核申请n个端口，这些端口从start开始，name 参数为设备的名称。如果返回NULL，则意味着申请端口失败。可以用来检查申请的资源是否可用，如果申请成功，则将其标志为已经使用，其他驱动再申请该资源时就会失败。

release_region()函数用来释放申请的IO资源。


```c
/*读写字节端口(8 位宽)*/
unsigned char inb(unsigned long port);
void outb(unsigned char byte, unsigned long port);

/*读写字端口(16 位宽)*/
unsigned short inw(unsigned long port);
void outw(unsigned short word, unsigned long port);

/*读写长字端口(32 位宽)*/
unsigned int inl(unsigned long port);
void outl(unsigned int longword, unsigned long port);

/*读写一串字节*/
void insb(unsigned long port, void *addr, unsigned long count);
void outsb(unsigned long port, void *addr, unsigned long count);


/*读写一串字*/
void insw(unsigned long port, void *addr, unsigned long count);
void outsw(unsigned long port, void *addr, unsigned long count);

/*读写一串长字*/
void insl(unsigned long port, void *addr, unsigned long count);
void outsl(unsigned long port, void *addr, unsigned long count);
```

`/proc/ioports`显示使用request_region( )分配的IO端口的情况。

## I/O内存

I/O 内存的访问步骤：首先是调用 request_mem_region()申请资源，接着将寄存器地址通过 ioremap()映射到内核空间虚拟地址，之后就可以通过 Linux 设备访问编程接口访问这些设备的寄存器了。访问完成后，应对 ioremap()申请的虚拟地址进行释放，并释放 release_mem_
region()申请的 I/O 内存资源。

![](.img/mem/io_mem.png)

```c
struct resource *request_mem_region(unsigned long start, unsigned long len, char *name);

void release_mem_region(unsigned long start, unsigned long len);

void __iomem *ioport_map(unsigned long port, unsigned int size);
void ioport_unmap(void __iomem *addr);

void __iomem *ioremap(unsigned long physaddr, unsigned long size);
void __iomem * ioremap_nocache (unsigned long offset, unsigned long size)
void iounmap(volatile void __iomem *addr);
```
ioremap()函数将设备所处的物理地址映射到虚拟地址。

ioremap_nocache()返回一个线性地址，此时CPU可以访问设备的内存空间。*phys_addr*要映射的物理地址 ；*size*要映射资源的大小。返回的映射地址必须使用`iounmap`来释放。

ioport_map()函数可以把 port 开始的 count 个连续的 I/O 端口重映射为一段内存空间，然后就可以在其返回的地址上像访问 I/O 内存一样访问这些 I/O 端口。

```c
/*读I/O内存*/
unsigned int ioread8(void __iomem *addr);
unsigned int ioread16(void __iomem *addr);
unsigned int ioread32(void __iomem *addr);

static inline unsigned char readb(const volatile void __iomem *addr)
{
	return *(volatile unsigned char __force *)addr;
}

static inline unsigned short readw(const volatile void __iomem *addr)
{
	return le16_to_cpu(*(volatile unsigned short __force *)addr);
}

static inline unsigned int readl(const volatile void __iomem *addr)
{
	return le32_to_cpu(*(volatile unsigned int __force *)addr);
}


/*写I/O内存*/
void iowrite8(u8 val, void __iomem *addr);
void iowrite8(u16 val, void __iomem *addr);
void iowrite8(u32 val, void __iomem *addr);


/*读一串 I/O 内存*/
void ioread8_rep(void __iomem *addr, void *buf, unsigned long count);
void ioread16_rep(void __iomem *addr, void *buf, unsigned long count);
void ioread32_rep(void __iomem *addr, void *buf, unsigned long count);

/*写一串 I/O 内存*/
void iowrite8_rep(void __iomem *addr, const void *buf, unsigned long count);
void iowrite16_rep(void __iomem *addr, const void *buf, unsigned long count);
void iowrite32_rep(void __iomem *addr, const void *buf, unsigned long count);


/*复制 I/O 内存*/
void memcpy_fromio(void *to, const volatile void __iomem *from, long count);
void memcpy_toio(volatile void __iomem *to, const void *from, long count);


/*设置 I/O 内存*/
void memset_io(volatile void __iomem *addr, unsigned char val, int count);
```

`/proc/iomem`文件记录物理地址的分配情况，这些地址范围是通过 requset_mem_region 函数申请得到的。



## DMA

DMA 是一种无需 CPU 的参与就可以让外设与系统内存之间进行双向数据传输的硬件机制。因为DMA的目的地址与Cache 所缓存的内存地址可能重叠，所以要禁止 DMA 目标地址范围内内存的Cache 功能。

DMA 操作在整个常规内存区域进行，DMA 的硬件使用总线地址而非物理地址。


