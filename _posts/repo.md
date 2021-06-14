<!--
 * @Description: 
 * @Author: luo_u
 * @Date: 2020-12-27 19:35:49
 * @LastEditTime: 2020-12-27 20:50:38
-->
[TOC]

repo是Google开发的用于管理Android版本库的一个工具。repo并不是用来取代Git，而是用Python对Git进行了一定的封装，简化了对多个Git版本库的管理。

Repo 分为两部分：第一部分是下载并安装的 repo 启动器。这是一个 Python 脚本，该脚本知道如何初始化检出，并可下载第二部分，即 Android 源代码检出中包含的完整 repo 工具。完整的 Repo 工具默认位于 `$SRCDIR/.repo/repo/` 中，它可以从下载的 Repo 启动器接收转发的命令。

### 安装

```shell
sudo apt-get install repo
```

```shell
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/.bin/repo
chmod a+rx ~/.bin/repo
```





### 初始化

```
repo init -u <URL> [<OPTIONS>]
```

在当前目录下安装 Repo。这会产生一个 .repo/ 目录，目录包括装 Repo 源代码和标准 Android 清单文件的 Git 仓库。.repo/ 目录还包括manifest.xml，是一个在.repo/manifests/目录选择清单的符号链接。

选项：

- -u 指定manifest仓库地址（URL）。
- -m 选择仓库中某个manifest文件，如果没有设置，就使用default.xml。
- -b 指定一个分支或修正版本。

### 同步

更新本地环境中的工作文件。如果不带参数，将同步所有项目的文件。

```
repo sync [<PROJECT_LIST>]
```

选项:
- -j <numbers>  多任务
- -c  只下载当前分支代码
- -d  让工程回退到manifest指定的版本
- -f  如果某个工程同步失败，继续同步
- -q  通过抑制状态消息来确保运行过程没有干扰。
- -s  同步到当前清单中的 manifest-server 元素指定的一个已知良好 build。

### 上传

```
repo upload [<PROJECT_LIST>]
```

在指定的项目中，Repo 把本地分支的更新比作远程分支在最后一次 Repo 同步。Repo 会提示你选择一个或更多尚未上传审查的分支。

### forall

```
repo forall [<PROJECT_LIST>] -c <COMMAND>
```

在每个项目中被给予的 shell 命令。如下的附加环境变量是通过 repo forall 才变得有效的：

- REPO_PROJECT 设置项目唯一的名称。
- REPO_PATH 是相对于客户端 root 的路径。
- REPO_REMOTE 是清单中远程系统的名称。
- REPO_LREV 是清单中修订本的名字，翻译成一个本地跟踪分支。如果你需要通过清单修正去本地执行 git 命令的时候可以使用。
- REPO_RREV 是清单中修订本的名字，正如在清单中所写的那样。

选项：

- -c  执行命令和参数。命令是通过 /bin/sh 评估的并且后面的任何参数就如 shell 位置的参数通过。
- -p  在指定命令的输出前显示项目标题。这是通过绑定管道到命令的stdin，stdout，和 sterr 流，并且用管道输送所有输出量到一个连续的流，显示在一个单一的页面调度会话中。
- -v  显示命令写到 sterr 的信息。



### 修改repo

应该在 .repo/manifests 文件夹里面修改repo的结构，然后用git命令提交。manifest.xml 文件结构:

```xml
<?xml version="1.0" encoding="UTF-8"?>  
<manifest>  

  <remote  name="shift"  
           fetch="git://git.mygit.com/" />  
  <default revision="kk-shift"  
           remote="shift"  
           sync-j="1" />  

  <project path="packages/shift/VideoPlayer" name="platform/packages/shift/VideoPlayer" />  
  <include name="another_manifest.xml" />
</manifest>  
```

- manifest
   这个是配置的顶层元素，即根标志。
- remote
   name：在每一个.git/config文件的remote项中用到这个name，即表示每个git的远程服务器的名字(这个名字很关键，如果多个remote属性的话，default属性中需要指定default remote)。git pull、get fetch的时候会用到这个remote name。
   alias ：可以覆盖之前定义的remote name，name必须是固定的，但是alias可以不同，可以用来指向不同的remote url
   fetch ：所有git url真正路径的前缀，所有git 的project name加上这个前缀，就是git url的真正路径
   review ：指定Gerrit的服务器名，用于repo upload操作。如果没有指定，则repo upload没有效果。
- default
   设定所有projects的默认属性值，如果在project元素里没有指定一个属性，则使用default元素的属性值。
   remote ：远程服务器的名字（上面remote属性中提到过，多个remote的时候需要指定default remote，就是这里设置了）
   revision ：所有git的默认branch，后面project没有特殊指出revision的话，就用这个branch
   sync_j ： 在repo sync中默认并行的数目
   sync_c ：如果设置为true，则只同步指定的分支(revision 属性指定)，而不是所有的ref内容
   sync_s ： 如果设置为true，则会同步git的子项目。
- manifest-server
   它的url属性用于指定manifest服务的URL，通常是一个XML RPC 服务
   它要支持一下RPC方法：
   GetApprovedManifest(branch, target) ：返回一个manifest用于指示所有projects的分支和编译目标。
   target参数来自环境变量`TARGET_PRODUCT`和`TARGET_BUILD_VARIANT`，组成`$TARGET_PRODUCT-$TARGET_BUILD_VARIANT`
   GetManifest(tag) ：返回指定tag的manifest。
- project
   需要clone的单独git
   name ：git 的名称，用于生成git url。URL格式是：${remote fetch}/${project name}.git 其中的 fetch就是上面提到的remote 中的fetch元素，name 就是此处的name
   path ：clone到本地的git的工作目录，如果没有配置的话，跟name一样
   remote ：定义remote name，如果没有定义的话就用default中定义的remote name
   revision ：指定需要获取的git提交点，可以定义成固定的branch，或者是明确的commit 哈希值
   groups ：列出project所属的组，以空格或者逗号分隔多个组名。所有的project都自动属于"all"组。每一个project自动属于
   name:'name' 和path:'path'组。例如<project name="monkeys" path="barrel-of"/>，它自动属于default, name:monkeys, and path:barrel-of组。如果一个project属于notdefault组，则，repo sync时不会下载
   sync_c ：如果设置为true，则只同步指定的分支(revision 属性指定)，而不是所有的ref内容。
   sync_s ： 如果设置为true，则会同步git的子项目
   upstream ：在哪个git分支可以找到一个SHA1。用于同步revision锁定的manifest(-c 模式)。该模式可以避免同步整个ref空间
   annotation ：可以有0个或多个annotation，格式是name-value，repo forall命令是会用来定义环境变量。
- include
   通过name属性可以引入另外一个manifest文件(路径相对与当前的manifest.xml 的路径)
   name ：另一个需要导入的manifest文件名字
   可以在当前的路径下添加一个another_manifest.xml，这样可以在另一个xml中添加或删除project。
- remove-project
   从内部的manifest表中删除指定的project。经常用于本地的manifest文件，用户可以替换一个project的定义。



### 快照

根据当前.repo的状态来创建一个配置文件，这个文件用来保存当前的工作状态。

```css
repo manifest -o snapshot.xml -r
```



恢复一个快照，可以用下面的命令：

```dart
cp snapshot.xml .repo/manifests/
repo init -m snapshot.xml
repo sync -d
```

注意：没有commit的修改不会恢复，已经commit的但是没有push的是可以恢复的，但只能从本地恢复。



### status

```
repo status [project-list]
```

对于每个指定的项目，将工作树与临时区域（索引）以及此分支 (HEAD) 上的最近一次提交进行比较。针对这三种状态之间存在差异的每个文件，显示其摘要行。

如需查看当前分支的状态，请运行 `repo status`。系统会按项目列出状态信息。对于项目中的每个文件，系统使用两个字母的代码来表示：

在第一列中，大写字母表示临时区域与上次提交状态之间的不同之处。

| 字母 | 含义       | 说明                                       |
| :--- | :--------- | :----------------------------------------- |
| -    | 没有变化   | 在 HEAD 与索引中相同                       |
| A    | 已添加     | 不存在于 HEAD 中，但存在于索引中           |
| M    | 已修改     | 存在于 HEAD 中，但索引中的文件已修改       |
| D    | 已删除     | 存在于 HEAD 中，但不存在于索引中           |
| R    | 已重命名   | 不存在于 HEAD 中，索引中文件的路径已更改   |
| C    | 已复制     | 不存在于 HEAD 中，复制自索引中的另一个文件 |
| T    | 模式已更改 | HEAD 与索引中的内容相同，但模式已更改      |
| U    | 未合并     | HEAD 与索引之间存在冲突；需要加以解决      |

在第二列中，小写字母表示工作目录与索引之间的不同之处。

| 字母 | 含义    | 说明                                       |
| :--- | :------ | :----------------------------------------- |
| -    | 新/未知 | 不存在于索引中，但存在于工作树中           |
| m    | 已修改  | 存在于索引中，也存在于工作树中（但已修改） |
| d    | 已删除  | 存在于索引中，但不存在于工作树中           |


```
repo update[ project-list ]
```
上传修改的代码 ，如果你本地的代码有所修改，那么在运行 repo sync 的时候，会提示你上传修改的代码。

```
repo diff [ project-list ]
```
显示提交的代码和当前工作目录代码之间的差异。

```
repo download  target revision
```
下载特定的修改版本到本地， 例如:  repo download pltform/frameworks/base 1241 下载修改版本为 1241 的代码

```
repo start branch-name [project-list]
```

从清单中指定的修订版本开始，创建一个新的分支进行开发。

- `branch-name` 参数用于说明您尝试对项目进行的更改。如果您不知道，请考虑使用名称 `default`。
- `project-list` 参数指定了将参与此主题分支的项目。



```
repo prune [project list]
```
删除已经merge 的 project