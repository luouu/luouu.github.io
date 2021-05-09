---
layout: cmd
cmd: diff3
topmost: false
---

diff3 - compare three files line by line，比较3个文件。

```shell
diff3 [OPTION]... MYFILE OLDFILE YOURFILE
```

##  选项

```shell
-a：把所有的文件都当做文本文件按照行为单位进行比较，即给定的文件不是文本文件；
-A：合并第2个文件和第3个文件之间的不同到第1个文件中，有冲突内容用括号括起来；
-B：与选项“-A”功能相同，但是不显示冲突的内容；
-e/--ed：生成一个“-ed”脚本，用于将第2个文件和第3个文件之间的不同合并到第1个文件中；
--easy-only：除了不显示互相重叠的变化，与选项“-e”的功能相同；
-i：为了和system V系统兼容，在“ed”脚本的最后生成“w”和“q”命令。此选项必须和选项“-AeExX3”连用，但是不能和“-m”连用；
--initial-tab：在正常格式的行的文本前，输出一个TAB字符而非两个空白字符。此选项将导致在行中TAB字符的对齐方式看上去规范。
```
