<!--
 * @Description: 
 * @Author: luo_u
 * @Date: 2020-05-18 11:49:24
 * @LastEditTime: 2020-06-01 14:52:36
--> 
[TOC]

# Makefile

下载地址：http://ftp.gnu.org/gnu/make/

## 语法
**makefile预定义变量**

| 预定义变量 |              作用              |
| :--------: | :----------------------------: |
|     AR     | 库文件维护程序的名称，默认为ar |
|     AS     |    汇编程序的名称，默认为as    |
|     CC     |    c编译器的名称，默认为cc     |
|    CXX     |   c++编译器的名称，默认为g++   |
|  ARFLAGS   |  库文件维护程序选项，无默认值  |
|  ASFLAGS   |     汇编程序选项，无默认值     |
|   CFLAGS   |     c编译器选项，无默认值      |
|  CXXFLAGS  |    c++编译器选项，无默认值     |

#### makefile自动变量

| 自动变量 |               作用               |
| :------: | :------------------------------: |
|    $*    |    不包含扩展名的目标文件名称    |
|    $<    |        第一个依赖文件名称        |
|    $?    | 所有时间戳比目标文件晚的依赖文件 |
|    $@    |        目标文件的完整名称        |
|    $^    |       所有不重复的依赖文件       |



####　自动变量

`$@`表示规则中的目标文件集。在模式规则中，如果有多个目标，那么就是匹配于目标中模式定义的集合。

`$%`仅当目标是函数库文件时，表示规则中的目标成员名，如果目标不是函数库文件（Unix 下是 .a ， Windows下是 .lib ），那么其值为空。

`$?`所有更新过的依赖目标的集合。以空格分隔。

`$^`所有的依赖目标的集合。以空格分隔。如果在依赖目标中有多个重复的，那个这个变量会去除重复的依赖目标，只保留一份。

`$<`依赖目标中的第一个目标名字。如果依赖目标是以模式（即 % ）定义的，那么 $< 将是符合模式的一系列的文件集。注意，其是一个一个取出来的。

`$+`所有依赖目标的集合，不去除重复的依赖目标。

`$*` 就是除了后缀的那一部分。如果目标中的后缀是make 所不能识别的，那么 $* 就是空值。



#### .PHONY） 
伪目标意思是这个目标本身不代表一个文件，执行这个目标不是为了得到某个文件或者东西，而是单纯为了执行这个目标下面的命令。 

###　subst
```makefile
$(subst 要被替换的字符串,用来替换的字符串,被处理的字符串)
```
`$(subst .c,.o,test1.c test2.c)`意思是用.o替换test1.c test2.c中的.c，最终得到test1.o test2.o。

###　wildcard
```makefile
$(wildcard 寻找的文件)
```
在系统中寻找文件，`$(wildcard *.c)`就等于找到系统中所有后缀为.c的文件，返回成以空格隔开的一整行字符。

### basename
```makefile
$(basename 文件名)
```
取得文件的名字（去掉后缀的意思），`$(basename test1.c)`就会取得test1。


##　特性
在makefile中使用shell命令，$var：将Makefile中的变量var展开，将其值传给shell命令。$$var：访问shell命令中定义的变量var，而非makefile的变量，如果某规则有多个shell命令行构成，而相互之间没有用';'和'\'连接起来的话，就是没有关联，相互之间也不能变量共享。


-　error: unused variable 'a' [-Werror=unused-variable]
   去掉编译配置的 -Werror
>LOCAL_CFLAGS += -Werror



