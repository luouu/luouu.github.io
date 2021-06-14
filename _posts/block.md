<!--
 * @Description: 块设备
 * @Author: luo_u
 * @Date: 2020-06-01 10:57:30
 * @LastEditTime: 2020-10-03 20:17:36
--> 

[TOC]

块设备是一种具有一定结构的随机存取设备，读写数据的基本单位为块，使用缓冲区来存放暂时的数据，待条件成熟后，从缓存一次性写入设备或者从设备一次性读到缓冲区。

- **页(page)**：Linux内存页，一般是4096字节，用作磁盘缓存。通过`getconf PAGE_SIZE`查看。
- **段(Segments)**：由若干个块组成。是Linux内存管理机制中一个内存页或者内存页的一部分。
- **块  (Blocks)** ：  Linux虚拟文件系统I/O操作的最小单位，通常为扇区的2的幂次方倍。通过`tune2fs -l /dev/vdb1|grep Block`查看块大小。
- **扇区(Sectors)**：块设备的基本单位。通常在512字节到32768字节之间，默认512字节。可以通过`fdisk -l`命令查看每个磁盘sector的大小



## 框架

Linux使用虚拟文件系统屏蔽不同I/O设备的差别，Linux内核通过映射层、通用块层、I/O调度层屏蔽不同块设备的差别。

![块设备框架](.img/block/block-arch.jpg)

![块设备框架](.img/block/block-arch2.jpg)

![块设备框架](.img/block/block.png)




## 块设备层

### gendisk

linux内核用 gendisk 结构体表示一个独立的磁盘设备或分区，用于对底层物理磁盘进行访问。

```c
struct gendisk {
	int major;			/* major number of driver */
	int first_minor;
	int minors;                     /* maximum number of minors, =1 for
                                         * disks that can't be partitioned. */

	char disk_name[DISK_NAME_LEN];	/* name of major driver */
	char *(*devnode)(struct gendisk *gd, mode_t *mode);

	unsigned int events;		/* supported events */
	unsigned int async_events;	/* async events, subset of all */

	struct disk_part_tbl __rcu *part_tbl;
	struct hd_struct part0;

	const struct block_device_operations *fops;
	struct request_queue *queue;
	void *private_data;

	int flags;
	struct device *driverfs_dev;  // FIXME: remove
	struct kobject *slave_dir;

	struct timer_rand_state *random;
	atomic_t sync_io;		/* RAID */
	struct disk_events *ev;
	int node_id;
};
```
```c
/* gendisk 是一个动态分配的结构体，需要内核来初始化，不能自己分配 */
struct gendisk *alloc_disk(int minors)

/* 注册磁盘设备 */
void add_disk(struct gendisk *disk)

/* 释放 gendisk */
void del_gendisk(struct gendisk *disk)
```

![块设备注册过程](.img/block/add_disk.png)



### block_device_operations
```c
struct block_device_operations {
	int (*open) (struct block_device *, fmode_t);
	int (*release) (struct gendisk *, fmode_t);
	int (*ioctl) (struct block_device *, fmode_t, unsigned, unsigned long);
	int (*compat_ioctl) (struct block_device *, fmode_t, unsigned, unsigned long);
	int (*direct_access) (struct block_device *, sector_t,
						void **, unsigned long *);
	unsigned int (*check_events) (struct gendisk *disk,
				      unsigned int clearing);
	/* ->media_changed() is DEPRECATED, use ->check_events() instead */
	int (*media_changed) (struct gendisk *);
	void (*unlock_native_capacity) (struct gendisk *);
	int (*revalidate_disk) (struct gendisk *);
	int (*getgeo)(struct block_device *, struct hd_geometry *);
	/* this callback is with swap_lock and sometimes page table lock held */
	void (*swap_slot_free_notify) (struct block_device *, unsigned long);
	struct module *owner;
};
```
- media_changed()检查可移动介质的驱动器中的介质是否已经改变，如果是返回一个非 0 值，否则返回 0。

- revalidate_disk()函数被调用来响应一个介质改变，使介质做好工作准备。

- getgeo()函数获得驱动器信息，根据驱动器的几何信息填充一个 hd_geometry 结构体，hd_geometry 结构体包含磁头、
扇区、柱面等信息，其定义于 include/linux/hdreg.h 头文件。

- owner是指向拥有这个结构体的模块的指针，通常被初始化为`THIS_MODULE`。



## 通用块层

通用块层 (Generic Block Layer) 负责维持一个 I/O 请求在上层文件系统与底层物理磁盘之间的关系。

### bio

bio 结构体用来对应一个 I/O 请求。

```c
struct bio {
	sector_t		bi_sector;	/* device address in 512 byte
						   sectors */
	struct bio		*bi_next;	/* request queue link */
	struct block_device	*bi_bdev;
	unsigned long		bi_flags;	/* status, command, etc */
	unsigned long		bi_rw;		/* bottom bits READ/WRITE,
						 * top bits priority
						 */

	unsigned short		bi_vcnt;	/* how many bio_vec's */
	unsigned short		bi_idx;		/* current index into bvl_vec */

	/* Number of segments in this BIO after
	 * physical address coalescing is performed.
	 */
	unsigned int		bi_phys_segments;
	unsigned int		bi_size;	/* residual I/O count */

	/*
	 * To keep track of the max segment size, we account for the
	 * sizes of the first and last mergeable segments in this bio.
	 */
	unsigned int		bi_seg_front_size;
	unsigned int		bi_seg_back_size;
	unsigned int		bi_max_vecs;	/* max bvl_vecs we can hold */
	unsigned int		bi_comp_cpu;	/* completion CPU */

	atomic_t		bi_cnt;		/* pin count */
	struct bio_vec		*bi_io_vec;	/* the actual vec list */
	bio_end_io_t		*bi_end_io;
	void			*bi_private;
#if defined(CONFIG_BLK_DEV_INTEGRITY)
	struct bio_integrity_payload *bi_integrity;  /* data integrity */
#endif

	bio_destructor_t	*bi_destructor;	/* destructor */

	/*
	 * We can inline a number of vecs at the end of the bio, to avoid
	 * double allocations for a small number of bio_vecs. This member
	 * MUST obviously be kept at the very end of the bio.
	 */
	struct bio_vec		bi_inline_vecs[0];
};
```

bio 描述了内存中连续几页的数据，每一页的数据由一个段 bio_vec 表示。多个页会被封装成多个段，这些段被组成一个数组，用 bio_io_vec 表示，bi_vcnt 是数组中元素的个数。

```c
struct bio_vec {
	struct page	*bv_page;
	unsigned int	bv_len;
	unsigned int	bv_offset;
};
```

```c
/* 遍历一个bio中的bi_io_vec 数组 */
int iter
struct bio_vec	 *bvec;
bio_for_each_segment(bvec, bio, iter)			

/* 返回I/O操作的数据方向 */
bio_rw(bio)

/* 结束I/O请求 */
void bio_endio(struct bio *bio, int error)
```



## I/O 调度层

I/O 调度层负责管理块设备的请求队列，对块设备提交的请求进行合并和排序预操作，以提高访问的效率。I/O调度算法可将连续的bio合并成一个请求，因此，一个请求可以包含多个bio，请求是bio经由I/O调度进行调整后的结果。

### request

在 Linux 中，驱动对块设备的输入或输出 (I/O) 操作，都会向块设备发出一个请求，在驱动中用 request 结构体描述。但对于一些磁盘设备而言请求的速度很慢，这时候内核就提供一种队列的机制把这些 I/O 请求添加到队列中（即请求队列），在驱动中用 request_queue 结构体描述。

```c
struct request {
	struct list_head queuelist;
	struct call_single_data csd;
	struct request_queue *q;
	unsigned int cmd_flags;
	enum rq_cmd_type_bits cmd_type;
	unsigned long atomic_flags;
	int cpu;

	/* the following two fields are internal, NEVER access directly */
	unsigned int __data_len;	/* total data len */
	sector_t __sector;		/* sector cursor */
	
	struct bio *bio;
	struct bio *biotail;
	
	struct hlist_node hash;	/* merge hash */
	/*
	 * The rb_node is only used inside the io scheduler, requests
	 * are pruned when moved to the dispatch queue. So let the
	 * completion_data share space with the rb_node.
	 */
	union {
		struct rb_node rb_node;	/* sort/lookup */
		void *completion_data;
	};
	
	/*
	 * Three pointers are available for the IO schedulers, if they need
	 * more they have to dynamically allocate it.  Flush requests are
	 * never put on the IO scheduler. So let the flush fields share
	 * space with the three elevator_private pointers.
	 */
	union {
		void *elevator_private[3];
		struct {
			unsigned int		seq;
			struct list_head	list;
		} flush;
	};
	
	struct gendisk *rq_disk;
	struct hd_struct *part;
	unsigned long start_time;
#ifdef CONFIG_BLK_CGROUP
	unsigned long long start_time_ns;
	unsigned long long io_start_time_ns;    /* when passed to hardware */
#endif
	/* Number of scatter-gather DMA addr+len pairs after
	 * physical address coalescing is performed.
	 */
	unsigned short nr_phys_segments;
#if defined(CONFIG_BLK_DEV_INTEGRITY)
	unsigned short nr_integrity_segments;
#endif

	unsigned short ioprio;
	int ref_count;
	void *special;		/* opaque pointer available for LLD use */
	char *buffer;		/* kaddr of the current segment if available */
	int tag;
	int errors;
	
	/*
	 * when request is used as a packet command carrier
	 */
	unsigned char __cmd[BLK_MAX_CDB];
	unsigned char *cmd;
	unsigned short cmd_len;
	unsigned int extra_len;	/* length of alignment and padding */
	unsigned int sense_len;
	unsigned int resid_len;	/* residual count */
	void *sense;
	unsigned long deadline;
	struct list_head timeout_list;
	unsigned int timeout;
	int retries;
	rq_end_io_fn *end_io;
	void *end_io_data;
	struct request *next_rq;
};
```

- 初始化请求队列
```c
typedef void (request_fn_proc) (struct request_queue *q);

struct request_queue *blk_init_queue(request_fn_proc *rfn, spinlock_t *lock)
```
rfn是请求处理函数的指针，lock是控制访问队列权限的自旋锁

- 清除请求队列
```c
void blk_cleanup_queue(struct request_queue *q)
```

- 分配请求队列
```c
struct request_queue *blk_alloc_queue(gfp_t gfp_mask)
```
对于 Flash、RAM 盘等完全随机访问的非机械设备，并不需要进行复杂的 I/O 调度，这个时候，应该使用上述函数分配一个请求队列，并使用如下函数来绑定请求队列和制造请求函数。
```c
typedef void (make_request_fn) (struct request_queue *q, struct bio *bio);

void blk_queue_make_request(struct request_queue *q, make_request_fn *mfn)
```

- 队列操作
```c
/* 从请求队列里面获得一个请求 */
struct request *blk_fetch_request(struct request_queue *q)

/* 判断是否是队列尾部 */
bool __blk_end_request_cur(struct request *rq, int error)

/* 获得请求数据传送方向，0 返回值表示从设备中读，非 0 表示向设备写 */
rq_data_dir(struct request *req);
```



### request_queue
```c
struct request_queue
{
	/*
	 * Together with queue_head for cacheline sharing
	 */
	struct list_head	queue_head;
	struct request		*last_merge;
	struct elevator_queue	*elevator;

	/*
	 * the queue request freelist, one for reads and one for writes
	 */
	struct request_list	rq;
	
	request_fn_proc		*request_fn;
	make_request_fn		*make_request_fn;
	prep_rq_fn		*prep_rq_fn;
	unprep_rq_fn		*unprep_rq_fn;
	merge_bvec_fn		*merge_bvec_fn;
	softirq_done_fn		*softirq_done_fn;
	rq_timed_out_fn		*rq_timed_out_fn;
	dma_drain_needed_fn	*dma_drain_needed;
	lld_busy_fn		*lld_busy_fn;
	
	/*
	 * Dispatch queue sorting
	 */
	sector_t		end_sector;
	struct request		*boundary_rq;
	
	/*
	 * Delayed queue handling
	 */
	struct delayed_work	delay_work;
	
	struct backing_dev_info	backing_dev_info;
	
	/*
	 * The queue owner gets to use this for whatever they like.
	 * ll_rw_blk doesn't touch it.
	 */
	void			*queuedata;
	
	/*
	 * queue needs bounce pages for pages above this limit
	 */
	gfp_t			bounce_gfp;
	
	/*
	 * various queue flags, see QUEUE_* below
	 */
	unsigned long		queue_flags;
	
	/*
	 * protects queue structures from reentrancy. ->__queue_lock should
	 * _never_ be used directly, it is queue private. always use
	 * ->queue_lock.
	 */
	spinlock_t		__queue_lock;
	spinlock_t		*queue_lock;
	
	/*
	 * queue kobject
	 */
	struct kobject kobj;
	
	/*
	 * queue settings
	 */
	unsigned long		nr_requests;	/* Max # of requests */
	unsigned int		nr_congestion_on;
	unsigned int		nr_congestion_off;
	unsigned int		nr_batching;
	
	void			*dma_drain_buffer;
	unsigned int		dma_drain_size;
	unsigned int		dma_pad_mask;
	unsigned int		dma_alignment;
	
	struct blk_queue_tag	*queue_tags;
	struct list_head	tag_busy_list;
	
	unsigned int		nr_sorted;
	unsigned int		in_flight[2];
	
	unsigned int		rq_timeout;
	struct timer_list	timeout;
	struct list_head	timeout_list;
	
	struct queue_limits	limits;
	
	/*
	 * sg stuff
	 */
	unsigned int		sg_timeout;
	unsigned int		sg_reserved_size;
	int			node;
#ifdef CONFIG_BLK_DEV_IO_TRACE
	struct blk_trace	*blk_trace;
#endif
	/*
	 * for flush operations
	 */
	unsigned int		flush_flags;
	unsigned int		flush_not_queueable:1;
	unsigned int		flush_queue_delayed:1;
	unsigned int		flush_pending_idx:1;
	unsigned int		flush_running_idx:1;
	unsigned long		flush_pending_since;
	struct list_head	flush_queue[2];
	struct list_head	flush_data_in_flight;
	struct request		flush_rq;

	struct mutex		sysfs_lock;

#if defined(CONFIG_BLK_DEV_BSG)
	struct bsg_class_device bsg_dev;
#endif

#ifdef CONFIG_BLK_DEV_THROTTLING
	/* Throttle data */
	struct throtl_data *td;
#endif
};
```

