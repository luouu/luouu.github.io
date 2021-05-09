---
layout: cmd
cmd: make
topmost: false
---

make - GNU make utility to maintain groups of programs，GNU的工程化编译工具。

```shell
make [OPTION]... [TARGET]...
```

## 选项

```shell
-f：指定“makefile”文件；
-i：忽略命令执行返回的出错信息；
-s：沉默模式，在执行之前不输出相应的命令行信息；
-r：禁止使用build-in规则；
-n：非执行模式，输出所有执行命令，但并不执行；
-t：更新目标文件；
-q：make操作将根据目标文件是否已经更新返回"0"或非"0"的状态信息；
-p：输出所有宏定义和目标文件描述；
-d：Debug模式，输出有关文件和检测时间的详细信息。
```

Linux下常用选项与Unix系统中稍有不同，下面是不同的部分：

```shell
-c dir：在读取 makefile 之前改变到指定的目录dir；
-I dir：当包含其他 makefile文件时，利用该选项指定搜索目录；
-h：help文挡，显示所有的make选项；
-w：在处理 makefile 之前和之后，都显示工作目录。
```
