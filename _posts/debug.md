<!--
 * @Description: 调试
 * @Author: luo_u
 * @Date: 2020-06-01 10:45:57
 * @LastEditTime: 2020-10-03 21:41:46
--> 

[TOC]

##　WARN_ON
内核要开启General setup->Configure standard kernel features->BUG() support

BUG_ON会引发oops，导致栈的回溯和错误信息的打印

```c
 #define BUG_ON(condition) do {
         if (unlikely((condition)!=0)) 
            BUG(); 
 } while(0)
```

WARN_ON则是调用dump_stack，打印堆栈信息，不会OOPS

```c
 #define WARN_ON(condition) do { 
       if (unlikely((condition)!=0)) { 
          printk("Badness in %s at %s:%d/n", __FUNCTION__, __FILE__,__LINE__); 
          dump_stack(); 
         } 
  } while (0)
```


## printk

```c
#define KERN_EMERG     "<0>" /* system is unusable */
#define KERN_ALERT     "<1>" /* action must be taken immediately */
#define KERN_CRIT      "<2>" /* critical conditions */
#define KERN_ERR       "<3>" /* error conditions */
#define KERN_WARNING   "<4>" /* warning conditions */
#define KERN_NOTICE    "<5>" /* normal but significant condition */
#define KERN_INFO      "<6>" /* informational */
#define KERN_DEBUG     "<7>" /* debug-level messages */
```

查看当前控制台的打印级别 `cat /proc/sys/kernel/printk`
  > 4    4    1    7 

分别表示：
- 控制台日志级别：优先级高于该值的消息将被打印至控制台;
- 默认的消息日志级别：将用该优先级来打印没有优先级的消息;
- 最低的控制台日志级别：控制台日志级别可被设置的最小值(最高优先级);
- 默认的控制台日志级别：控制台日志级别的缺省值;

修改打印
>echo "新的打印级别  4    1    7" >/proc/sys/kernel/printk

打印级别不够的信息会被写到日志中，可通过`dmesg`命令来查看。



## Sparse

Sparse通过 gcc 的扩展属性 __attribute__ 以及自己定义的 __context__ 来对代码进行静态检查。

| **宏名称**       | **宏定义**                                 | **检查点**                                                   |
| ---------------- | ------------------------------------------ | ------------------------------------------------------------ |
| __bitwise        | __attribute__((bitwise))                   | 确保变量是相同的位方式(bit-endian, little-endiandeng)        |
| __user           | __attribute__((noderef, address_space(1))) | 指针地址必须在用户地址空间初始化，不能指向内核地址空间, 设备地址空间 |
| __kernel         | __attribute__((noderef, address_space(0))) | 指针地址必须在内核地址空间初始化                             |
| __iomem          | __attribute__((noderef, address_space(2))) | 指针地址必须在设备地址空间初始化                             |
| __safe           | __attribute__((safe))                      | 变量可以为空                                                 |
| __force          | __attribute__((force))                     | 变量可以进行强制转换                                         |
| __nocast         | __attribute__((nocast))                    | 参数类型与实际参数类型必须一致                               |
| __acquires(x)    | __attribute__((context(x, 0, 1)))          | 参数x 在执行前引用计数必须是0，执行后，引用计数必须为1       |
| __releases(x)    | __attribute__((context(x, 1, 0)))          | 与 __acquires(x) 相反                                        |
| __acquire(x)     | __context__(x, 1)                          | 参数x 的引用计数 + 1                                         |
| __release(x)     | __context__(x, -1)                         | 与 __acquire(x) 相反                                         |
| __cond_lock(x,c) | ((c) ? ({ __acquire(x); 1; }) : 0)         | 参数c 不为0时,引用计数 + 1, 并返回1                          |

其中 `__acquires(x)`  和`__releases(x)`，  `__acquire(x)`和  `__release(x)` 必须配对使用, 否则 Sparse 会给出警告。



