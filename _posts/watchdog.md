<!--
 * @Description: watchdog
 * @Author: luo_u
 * @Date: 2020-06-01 10:45:05
 * @LastEditTime: 2020-07-26 17:30:48
--> 

[TOC]

- watchdog_dev.c
- watchdog_core.c
- wdt.c


内核配置`CONFIG_WATCHDOG_NOWAYOUT`选项表示看门狗一旦开启就无法被停止，可设置nowayout模块参数。

如果驱动支持"Magic Close"，除非在关闭看门狗前，魔幻字符'V'发送到/dev/watchdog，否则不能停止看门狗。