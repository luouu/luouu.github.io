---
layout: post
title: c语言数据类型
categories: [c]
description: 
keywords: 
topmost: false
---

* TOC
{:toc}

| Data Type |                         ILP32                          |   LP32    | ILP64 |                   LP64                   |   LLP64   |
| :-------: | :----------------------------------------------------: | :-------: | :---: | :--------------------------------------: | :-------: |
|  宏定义   |                           _                            |     _     |   _   |                 __LP64__                 | __LLP64__ |
|   平台    | Win32 API  / Unix 和 Unix 类的系统 （Linux，Mac OS X） | Win16 API |       | Unix 和 Unix 类的系统 （Linux，Mac OS X) | Win64 API |
|   char    |                           8                            |     8     |   8   |                    8                     |     8     |
|   short   |                           16                           |    16     |  16   |                    16                    |    16     |
|    int    |                           32                           |    16     |  64   |                    32                    |    32     |
|   long    |                           32                           |    32     |  64   |                    64                    |    32     |
| long long |                           64                           |    64     |  64   |                    64                    |    64     |
|  pointer  |                           32                           |    32     |  64   |                    64                    |    64     |
|   float   |                           32                           |           |       |                    32                    |           |
|  double   |                           64                           |           |       |                    64                    |           |

1. 格式化打印，long使用%ld或%lx，指针使用%p
2. 对于long类型，常量要加L，如：long a = 1L<<32
3. ssize_t在32位机器上等同与int，在64位机器上等同与long，size_t是无符号型的ssize_t。
4. 指针+1 = p + sizeof(p指向的数据类型)。sizeof(p)在32位机器上永远是4字节，64位机器上是8字节。

stdint.h 里定义了一些数据类型的别名和范围。

```c
typedef signed char    int8_t;
typedef short int      int16_t;
typedef int            int32_t;
# if __WORDSIZE == 64
typedef long int       int64_t;
#else
__extension__
typedef long long int    int64_t;
# endif

/* Unsigned.  */
typedef unsigned char        uint8_t;
typedef unsigned short int   uint16_t;
typedef unsigned int         uint32_t;
#if __WORDSIZE == 64
typedef unsigned long int   uint64_t;
#else
__extension__
typedef unsigned long long int  uint64_t;
#endif

/* Types for `void *' pointers.  */
#if __WORDSIZE == 64
typedef long int            intptr_t;
typedef unsigned long int   uintptr_t;
#else
typedef int                 ntptr_t;
typedef unsigned int        uintptr_t;
#endif

/* Limits of integral types.  */

/* Minimum of signed integral types.  */
# define INT8_MIN        (-128)
# define INT16_MIN       (-32767-1)
# define INT32_MIN       (-2147483647-1)
# define INT64_MIN       (-__INT64_C(9223372036854775807)-1)
/* Maximum of signed integral types.  */
# define INT8_MAX        (127)
# define INT16_MAX       (32767)
# define INT32_MAX       (2147483647)
# define INT64_MAX       (__INT64_C(9223372036854775807))

/* Maximum of unsigned integral types.  */
# define UINT8_MAX        (255)
# define UINT16_MAX       (65535)
# define UINT32_MAX       (4294967295U)
# define UINT64_MAX       (__UINT64_C(18446744073709551615))
```
