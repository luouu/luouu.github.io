
# gcc

[TOC]

## 选项

```shell
-o：指定生成的输出文件；
-E：仅执行编译预处理；
-S：将C代码转换为汇编代码；
-wall：显示警告信息；
-c：仅执行编译操作，不进行连接操作。
```

## 参数

- **-rdynamic**

连接选项 ，指示连接器把所有符号都添加到动态符号表（.dynsym）里，以便那些使用符号表的函数使用，如dlopen() 或 backtrace()。

- **-g**

收集调试信息并将存放可执行文件。

- **-fsanitize=address**

gcc从4.8版本起，集成了Address Sanitizer工具，可以用来检查内存访问的错误，编译时指定`-fsanitize=address -g`，可以检测的问题有：

- 内存泄漏
- 堆栈和全局内存越界访问
- free后继续使用
- 局部内存被外层使用
- Initialization order bugs

- **-static**

此选项将禁止使用动态库

- **-share**

此选项将尽量使用动态库

### 依赖文件

-M
生成文件的依赖关系，依赖项是源文件中引用的所有头文件，同时也把一些标准库的头文件包含了进来。

-MM
生成文件的依赖关系，和 -M 类似，但不包含标准库的头文件。

-MG
要求把缺失的头文件按存在对待，并且假定他们和源文件在同一目录下，必须和`-M`选项一起用。

-MF [file]
当使用了-M或-MM选项时，则把依赖关系写入名为file的文件中。若同时也使用了-MD或-MMD,-MF将覆写输出的依赖文件的名称 。

-MD
等同于 -M -MF file，但是默认关闭了 -E 选项。其输出的文件名是基于 -o 选项，若给定了 -o 选项，则输出的文件名是 -o 指定的文件名，并添加 .d 后缀，若没有给定，则输入的文件名作为输出的文件名，并添加 .d 后缀，同时继续指定的编译工作。

-MMD
类似于-MD”，但是输出的依赖文件中，不包含标准头文件。

-MP
生成的依赖文件里面，依赖规则中的所有 .h 依赖项都会在该文件中生成一个伪目标，其不依赖任何其他依赖项。该伪规则将避免删除了对应的头文件而没有更新Makefile去匹配新的依赖关系而导致 make 出错的情况出现。

-MT Target
在生成的依赖文件中，指定依赖规则中的目标。
