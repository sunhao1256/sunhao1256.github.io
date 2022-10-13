---
title: linux
date: 2020-01-31 20:50:54
tags: ['linux']
---

# 终端

[root@LocalHost 桌面]# l

- root表示当前登录用户名
- locahost表示当前登录的主机名
- 桌面 表示当前的工作目录
- \# 身份标识符 \#表示为超级管理员 $表示为普通用户

# 目录结构

- Bin：全称Binary，存放的都是一些二进制文件
- Dev：主要存放一些外接设备，例如U盘等，在其中的设备是不能直接使用的，需要挂载（类似windows下的分盘）
- Etc：主要存放一些配置文件
- Home：表示除了root用户以外的其他的家目录，类似windows下的user用户目录
- Proc：该目录存放运行的进程文件
- Root：该目录是root用户的家目录
- SBin：该目录也是存放二进制文件，但是必须得有super权限的用户才能执行
- Usr：用户的应用程序和文件都放在这个目录下类似于windows下的program files目录
- Mnt：让用户临时挂在别的文件系统 
- Opt：一般存放安装软件包
- Usr/local：存放安装软件后存在的软件目录
- Var：存放不断变化的文件，例如日志文件

# VI和VIM编辑器

所有的Linux系统都会内建VI文本编辑器，VIM具有程序编辑的能力，可以看成是VI的增强版

## VI和VIM的3种常见模式

- 正常模式

  在正常是模式下，可以使用快捷键来处理内容

- 插入/编辑模式

  在此模式下，可以输入内容，按i，I，o，O，a，A，r，R等任何一个字母就可以进入编辑模式，一般用i

- 命令行模式

  可以使用指令完成，读取，存盘，替换，离开，显示行号等动作

## 注释

单行注释：#

多行注释：**:<<!内容!**

## 快捷键的使用

- 行首0，行尾$

- 拷贝当前行：yy，拷贝当前向下n行：nyy，例如：5yy

- 粘贴：p

- 删除当前行：dd，删除当前行下n行：ndd，例如：5dd

- 查找：命令行模式下 / 关键字，例如：/hello，下一个n

- 设置文件行号：命令行模式下 ：set nu，关闭行号：set nonu

- 快速到顶行：正常模式下 gg

- 快速到底行：正常模式下G

- 撤销：正常模式下u

- 指定到行号：先显示行号，在正常模式下先输入行数，再按shift+g

- o：进入输入模式，且进入下一行

- 复制，删除/剪切
yy：复制光标所在一行
3yy：复制光标所在行及向下2行
p：粘贴
dd：剪切/删除光标所在一行
3dd：剪切/删除光标所在行及向下两行
D：从当前的光标开始剪切一直到行末
d0：从当前光标开始剪切一直到行首
x：删除当前的光标，每次只会删除一个
X：删除当前光标前面的那个，每次只会删除一个
控制光标上下左右走：h左j下k上l右

- 定位：
H：当前屏幕的上方
M：当前屏幕的中间
L：当前屏幕的下方
Ctrl+f：向下翻一页代码
Ctrl+b：向上翻一页代码
Ctrl+d：向下翻半页代码
Ctrl+u：向上翻半页代码
20G：快速的定位到第20行，定位到第1行就是1G，以此类推
G：快速的回到整个代码的最后一行
gg：快速回到整个代码的第一行
w：向后跳一个单词的长度，即调到下一个单词的开始处
b：向后跳一个单词的长度，即调到上一个单词的开始处

- 撤销
u：撤销刚刚的动作
Ctrl+r：反撤销

- 选中一片代码
  v：选中一片代码
  V：选中一片代码

  ：向右移动代码

  <<：向左移动代码

  . ：重复执行上一次的命令

- 替换
  r：替换一个字符
  R：替换光标以及后面的字符
  shift+zz：保存并退出

- 末行模式下
  替换
  将当前文件中的所有abc替换成123
  :%s/abc/123/g
  将第一行到第十行之间的abc替换成123
  :1, 10s/abc/123/g**
  查找
  查找整个代码中的hello
  ：/hello
  n：下一个
  N：上一个
  保存和退出
  :q 退出不保存
  :wq 保存并退出
  :q ! 强制退出不保存
  :wq! 强制保存并退出

# 关机和重启

- shutdown -h now ：表示立即关机
- shutdown -h 1：表示1分钟后关机
- shutdown -r now：表示立即重启
- halt：直接关机
- reboot：重启系统
- sync：把内存数据同步到磁盘上，防止数据丢失

# 登录和注销

- 一般是使用普通账号进行登录，当权限不够的时候在用"su - 用户名"切换到系统管理员身份
- logout注销用户

# 用户管理

每个用户至少存在一个用户组中。

- 添加用户： useradd [选项] 用户名，如果不指定用户组会给用户创建一个同名的用户组，并存放在其中。

  useradd -d 路径 用户名，指定用户的访问路径

  useradd -g 组名 用户名，指定用户的用户组

- 设置密码： passwd 用户名

- 查看当前目录路径： pwd

- 删除用户： userdel 用户名，userdel -r 用户名，删除用户且删除家目录，（一般不删除家目录）

- 查询用户信息： id 用户名

- 切换到其他用户： su -用户名

- ```go
  * `/etc/passwd`： 用户账户的详细信息在此文件中更新。
  * `/etc/shadow`： 用户账户密码在此文件中更新。
  * `/etc/group`： 新用户群组的详细信息在此文件中更新。
  * `/etc/gshadow`： 新用户群组密码在此文件中更新。
  ```

# 用户组管理

- 新增：groupadd 组名
- 删除：groupdel 组名
- 修改用户的用户组：usermod -g 组名 用户名，修改用户的用户组
- 修改用户的登录目录：usermod -d 路径名 用户名

# 指定运行级别

系统的运行级别文件在/etc/inittab，可以设置默认的运行级别

- 0：关机

- 1：单用户

- 2：多用户无网络服务

- 3：多用户有网络服务

- 4：保留

- 5：图形化界面

- 6：重启

  切换到指定运行级别命令：init(级别)

  注意点：如果root密码丢失的话，可以指定运行级别为1，单用户级别，直接免密码登录root，然后修改密码。

# 帮助指令

man 命令

# 文件目录类指令

- pwd： 显示当前工作的绝对路径

- ls： 查看当前目录结构

  ls -[选项]

  -a：查出所有加上隐藏文件

  -l：横向展示文件信息

  -h：以人类的方式查看，可以显示文件具体大小单位

- cd： 切换到其他目录

- mkdir： 创建目录 mkdir[选项] 目录名

  mkdir -p 目录名，创建多级目录

- rmdir：删除一个空目录（无法删除非空目录，需要使用rm -rf 目录名，才可以进行删除）

-  touch：创建一个或多个空文件

- cp：拷贝文件到指定目录  

  cp 文件名 目标路径

  \cp ：如果有存在文件，强制覆盖

  cp -r 目录名 目标路径 ：递归拷贝整个文件夹

- rm：移除文件或目录

  rm -r：递归删除整个文件夹

  rm -f：强制删除不提示

- mv：移动文件与重命名

  mv a.txt b.txt 重命名

  mv /a /b 移动文件

- cat：浏览文件，只读，不能修改，一般配合 |more分页显示

  cat -n 文件名 |more  分页显示行号浏览文件

- more：以全屏方式去显示文件内容，空格下一页，enter下一行，ctrl+f下一页，ctrl+b上一页

- less：与More不同的是，less不是一次将文件加载完才显示，所以对大型文件效率高，推荐查看日志文件

- \> ：输出重定向，覆盖原先内容（文件不存在就创建）

  echo 'test' test.txt

- \>>：追加指令

- cal ：显示当前日历

- echo：输入内容到控制台，经常输出环境变量

  echo $PATH

- head：用于显示文件的开头部分

  head -n 5 文件名：查看文件的前5行，默认10行

- tail：查看文件尾部输出内容

  tail 文件 查看文件最后10行

  tail -n 100 查看文件最后100行

  tail -f 实时输出文件

- ln：软连接，类似windows的快捷方式

  ln -s [原文件或者目录] [软连接名]（用pwd查看的时候，依然是软连接的所在的目录）

- history：查看历史使用过的指令

  history n：查看最后的n条

  !188：执行历史的188号指令

# 时间日期类指令

- date：显示当前日期时间

  date：显示当前时间

  date +%Y：显示当前年份

  date +%m：显示月份

  date +%M：显示分

  date +‘%Y/%M’：显示2020/*

  date -s 字符串时间：设置系统实际

- cal：以日历的方式显示时间

# 搜索查找类指令

- find：从指定目录向下递归遍历各个子目录，将满足条件的文件或者目录显示在终端

  find [搜索范围] [选项]

  find /root -name test.txt：在root下查找test.txt文件，可以使用*做模糊查询

  find /root -user root：在root下查找是root用户创建的文件

  find /root -size +20M：在root下查找大于20M的文件 ， +大于，-小于，不写是等于

- locate：可以快速定位文件路径，locate无需遍历整个文件系统，但必须定期更新locate数据库

  使用updatedb

- grep 和 | 管道指令

  grep过滤查找，管道符 | 表示将前面一个命令的处理结果交给后面的命令处理

  grep基本用法：grep[选项] 查找内容 原文件

  grep常用选项：-n 显示匹配行以及行号 -i 忽略大小写

  例如： cat /root/testLog.log | grep  -ni 300465

# 压缩和解压缩

- gzip和gunzip

  gizp 文件（只能将文件压缩为*.gz文件，且不保留原文件）

  gunzip 文件.gz（解压文件，也不保留压缩文件）

- zip和unzip

  zip  [选项] xxx.zip 要压缩的内容（压缩文件，保留原文件）

  - -r 递归压缩，压缩整个目录

  unzip [选项] xxx.zip （解压文件，保留压缩文件）

  - -d 目录地址 ，具体解压到哪

- tar

  tar是指打包指令，最后打包的指令是.tar.gz文件

  tar [选项] xxx.tar.gz 压缩内容

  - -c 产生.tar打包文件
  - -v 显示详细信息
  - -f 指定压缩后的文件名
  - -z 打包同时压缩
  - -x 解压.tar文件
  - -C 解压到指定目录下
- -P tar默认是相对路径，可以使用P设置为绝对路径
  

压缩例如：tar -zcvf a.tar.gz test.txt test1.txt ；tar -zcvf a.tar.gz /home/

  解压例如：tar -zxvf a.tar.gz -C /

- 解压rar包
- 解压tar包

# 组管理和权限管理

在Linux中的每一个用户都必须属于一个组，不能独立于组外，在linux中每个文件都有所有者、所在组、其他组的概念

- 文件/目录的所有者（谁创建了文件，就成为该文件的所有者）

  查看所有者 ls -ahl 

- 修改文件所有者

  chown 用户名 文件名

- 修改文件所在组

  chgrp 组名 文件名

## 权限基本介绍

一个文件或目录的详细描述

例如：

```
-rw-r--r-- 1 root root         0 Feb  1 18:39 a.txt
```

**0-10**位说明：

- 0位表示文件类型

  - \- ：普通文件

  - d：目录

  - l：链接文件，软连接（快捷方式）

  - c：字符设备（键盘，鼠标）

  - b：块文件（硬盘）

- 1-3位表示文件所有者拥有的权限

- -6位表示文件所在组拥有的权限

- 7-9位表示文件其他组用户拥有的权限

- 10位表示 如果是文件，则表示文件的硬连接数，如果是目录表示子目录个数

**rwx**权限详解

- rwx作用在文件上
  - r：表示可读，可以查看
  - w：表示修改，可以修改，但不代表可以删除该文件，删除的前提条件是拥有文件所在目录写的权限，才可以删除
  - x：表示可以执行，表示可执行
- rwx作用在目录上
  - r：表示可读取，ls查看目录内容
  - w：表示可写，可以修改，目录内创建+删除+重命名目录
  - x：表示可执行，可以进入该目录

- rwx用数字表示

  - r：4
  - w：2
  - x：1

  因此rwx=7

  ---

  

## 权限修改chmod

可以通过chmod修改文件或目录的权限

- 通过+ 、- 、= 来修改权限

  - u：表示所有者

  - g：表示所在组

  - o：表示其他人

  - a：表示所有人（o，g，u的和）

    例如：chmod u=rwx,g=rx,o=x 文件目录名，chmod u-x 文件目录名（让所有者失去x执行权限）,chmod u+x（让所有者加上x执行权限）

- 通过数字变更权限

  - r：4

  - w：2

  - x：1

    例如：chmod u=rwx,g=rx,o=x 文件名  **相当于**   chmod 751 文件名

- 修改文件所有者-chown

  - chown 新用户 文件名（修改文件的所有者）
  - chown 新用户:新组 文件名（修改文件的所有组和所有者）
  - -R 如果是目录，则使其下所有的子文件或者目录递归生效

# crond任务调度

命令：crontab[选项]

<p style="color:red;font-weight:700">Linux的crond调度任务最小单位为分！</p>
选项：

- -e：编辑crontab 定时任务

- -l：查询crontab 任务

- -r：删除当前用户的所有crontab 任务

- service crond restart ：(重启任务调度)

  可以写一个sheel脚本，然后去执行

# 分区与挂载

## 分区的方式

- mbr分区
  - 最多支持4个主分区
  - 系统只能安装在主分区
  - 扩展分区占用一个主分区
  - mbr最大只支持2TB，但有最好的兼容性
- gtp分区
  - 支持无限多个主分区（但操作系统下可能会限制，例如windows下最多支持128个分区）
  - 最大支持18EB的容量（EB=1024PB，PB=1024TB）
  - windows7 64位以后支持gtp
- lsblk 可以查看当前分区情况

## 挂载

- 增加一块新硬盘
- 分区  fdisk /dev/硬盘名
- 格式化 mkfs -t 文件类型 分区路径。例如：mkfs -t ext4 /dev/sdb1
- 挂载 mount 分区路径 目标目录（只是临时挂载）
- 设置自动挂载：修改 /etc/fstab文件，增加自己的分区挂载到目标目录上，再使用mount -a 自动挂载

- 卸载 umount 分区路径

# 磁盘情况查询

- df -h，df -l

- du 查看某个目录下的磁盘使用情况

  可带参数

  - -c 列出明细同时提供总数
  - --max-depth=n 子目录深度为n
  - -a 查所有包含文件
  - -h 带单位

  例如：du -cha --max-depth=1 /root

## 常用命令

- 统计目录下文件的个数 ls -al |grep '^-' | wc -l
- 统计目录下目录的个数 ls -al |grep '^d' | wc -l
- 统计目录下包含子目录文件的个数 ls -alR |grep '^-' | wc -l
- 查看目录树 tree

# 网络配置

- 修改linux ip地址为指定地址

  修改/etc/sysconfig/network-scripts/ifcfg-eth0 

---



# 进程管理

- 基本介绍

  **ps**命令是用来查看目前系统中有哪些正在执行，以及他们的执行状况

  可以配合grep使用

  常用选项

  - -a 查看当前终端所有的进程信息
  - -u 以用户的格式显示进程信息
  - -x 显示后台进程运行的参数
  - -ef 展示父进程

  显示字段

  - USER 用户名称

  - PID 进程识别号
  - %CPU 进程占用CPU百分比
  - %MEM进程占用内存百分比
  - VSZ 进程占用虚拟内存大小 （kb）
  - RSS 进程占用物理内存大小 （kb）
  - TTY 终端机号
  - STAT 进行状态，s表示进程是绘画的先导进程，N表示进程拥有比普通进程优先级更低的优先级，R表示正在运行，D表示短期等待，Z表示僵尸进程，T表示被追踪或者被停止
  - TIME 此进程所消耗CPU时间
  - CMD 正在执行的命令或进程名 

- 终止进程kill和killAll

  - kill [选项] 进程号

    常用选项

    - -9 表示强迫进程终止
    - 

  - kill 进程名(通过名称终止进程，可以使用通配符)

- 查看进程树 pstree

  -p显示进程ID，-u显示进程所在用户

---

# 服务管理

服务（Service）本质上就是一个进程，通常运行在后台，会监听一个端口，等待其他程序的请求，例如mysqld,sshd,防火墙等，因此我们又称为守护进程

## service管理命令

service 服务名 start|stop|restart|reload|status

**在Centos7.0以后使用systemctl**

- ls -l /etc/init.d/ 查看系统有哪些服务

- chkconfig可以给每个服务设置各个运行级别自启动/关闭
  - 开启或关闭：chkconfig --level 5 服务名 on/off

# 进程监控

**top**命令与ps不同之处在于，top会不停的更新数据

常用选项

- -d 秒数 ：几秒后更新 默认3秒
- -i ：不显示闲置或僵尸进程
- -p 进程号：显示进程号
- -u 用户名：监控指定的用户

交互操作

- P：按cpu使用率排序

- M：按内存使用排序
- N：按PID排序
- q：退出top

# 网络监控

**netstat**用来监控网络

常用选项

- -an ：按照一定顺序排列
- -p：显示那个进程在使用

# Lsof管理

**lsof**是系统管理的工具，lsof可以代替ps和netstat的几乎全部工作

## 关键选项

- 默认 : 没有选项，lsof列出活跃进程的所有打开文件

- 组合 : 可以将选项组合到一起，如-abc，但要当心哪些选项需要参数

- -a : 结果进行“与”运算（而不是“或”）

- -l : 在输出显示用户ID而不是用户名

- -h : 获得帮助

- -t : 仅获取进程ID

- -U : 获取UNIX套接口地址

- -F : 格式化输出结果，用于其它命令。可以通过多种方式格式化，如-F pcfn（用于进程id、命令名、文件描述符、文件名，并以空终止）

- -i：显示所有连接（**语法：lsof -i[46] [protocol][@hostname|hostaddr][:service|port]**）

  例如：

  - lsof  -i 6：获取IPV6流量
  - lsof -i :3306：查看3306端口使用状态

## 参考连接

https://www.jianshu.com/p/a3aa6b01b2e1

# RPM包管理

rpm -ql |grep 查询内容

常用选项

- -i ：安装
- -v：提示
- -h：进度条
- -e：卸载RPM包（加上--nodeps强制删除）
- -q：查询包

rpm -ivh RPM包全路径名称

 rpm -ivh –relocate = 路径

# Yum

**Yum**可以自动处理RPM依赖关系

基本指令

- list：查看
- install：安装

# Shell脚本

## 起步

- 新建一个shell脚本，一般以.sh结尾
- 写入内容，保存
- 授予x权限
- 执行

##  shell变量

### 介绍

shell变量分为系统变量和用户自定义变量

- 系统变量：$HOME,$PWD,$SHELL,$USER,"$PATH"等等，比如echo $HOME
- 显示当前shell中所有的变量：set

### 定义变量

#### 基本语法

- 定义变量：变量=值
- 撤销变量：unset 变量
- 声明静态变量：readonly 变量，注意：不能unset

#### 定义规则

- 变量名称可以由字母，数字和下划线组成，但是不能以数字开头
- 等号两侧不能有空格
- 变量名称习惯性是大写

#### 将命令的返回值赋值给变量

- A=$(ls -la)
- A=\`ls -la` 使用反引号（数字键左边的键）

#### 设置环境变量

- export 变量名=变量值（功能描述：将shell变量输出为环境变量）
- source 配置文件 （让修改后的配置文件立即生效）
- echo $变量名 （查询环境变量的值）

#### 设置位置参数变量

例如 ./myshell.sh 100 200 我们可以在myshell脚本中获取到100,200的参数信息

基本语法：

- $n：n为数字，$0表示命令本身，$(1-9)代表第一到第九个参数，十以上的参数需要用大括号 ${10}
- $*：这个变量命令表示所有参数，将所有参数看成一个整体
- $@：也是表示所有参数，但把参数区分开来
- $#：表示所有参数的个数

#### 预定义变量

- $$：当前进程的PID
- $!：后台运行的最后一个进程的进程号
- $?：最后一次执行命令的状态，如果成功为0，如果不成功为开发者指定的一个值
- 以后台方式运行：在命令最后使用一个&

#### 运算符

基本语法：

- $((运算式))或$[运算式]

- expr m + n（注意运算符间有空格）

- expr m - n 

- expr \\*，/，%：乘除取余

  **expr需要用反引号**

#### 条件判断

- 基本语法

  [ condition ] （注意condition前后要有空格）

  非空返回true ，可以使用$?验证（0true，>1false）

- 常用判断

  - 两个整数比较
    - =字符串比较
    - -lt小于
    - -le小于等于
    - -eq等于
    - -gt大于
    - -ge大于等于
    - -ne不等于
  - 按照文件读写权限判断
    - -r有读
    - -w有写
    - -x有执行
  - 按照文件类型判断
    - -f文件存在且是一个常规文件
    - -e文件存在
    - -d文件存在且是一个目录
  
  示例：
  
  ```shell
  #!/bin/bash
  if [ $1 -ge 60 ]
  then
          echo "及格啦"
  elif [ $1 -ge 40 ]
  then    echo "你废物"
  
  fi
  
  ```
  
  

#### case语句

基本语法示例：

```shell
#!/bin/bash
case $1 in
"1")
echo "周一"
;;
"2")
echo "周二";;
*)
echo "other";;
esac

```

#### for语句

- for 变量 in 值1 值2 值3...

  do 

  ​	程序

  done

  示例：

  ```shell
  #!/bin/bash
  # 使用$*
  for i  in "$*"
  do
          echo "$i"
  done
  
  #使用$@
  for i in $@
  do
          echo "param=$i"
  done
  ```

  

  

- for((初始值；循环控制条件；变量变化))

  do

  ​	程序

  done

  示例：

  ```shell
  #!/bin/bash
  SUM=0
  for ((i=1;i<=100;i++))
  do
          SUM=$[$SUM + $i ]
  done
  echo $SUM
  ```

#### while循环

while [ condition ]

do

​	程序

done

示例：

```shell
#!/bin/bash
RESULT=0
I=0
while [ $I -le $1 ]
do
        RESULT=$[ $RESULT + $I ]
        I=$[ $I + 1 ]
done
echo $RESULT
```

#### read读取控制台输入

基本语法

**read [选项] 参数**

- -p：指定读取时的提示符；
- -t：指定读取值时的等待时间 

示例：

```shell
read -p "请输入一个数字" NUM
echo $NUM
```

#### 函数

- 系统函数

  - basename [pathname]\[suffix] (常用于获取文件名)
  - dirname （常用于返回路径名称）

- 自定义函数

  基本语法

  [ function ] functionname [()]{

  ​	Action;

  ​	[return int]

  }

  示例：

  ```shell
  #!/bin/bash
  # 计算输入的2个数字和
  read -p "请输入第一个数字" NUM1
  read -p "请输入第二个数字" NUM2
  
  function getSum(){
          SUM=$[ $NUM1 + $NUM2 ]
          echo "和是:$SUM"
  }
  
  getSum $NUM1 $NUM2
  ```

## 综合案例

### 需求：

- 每天凌晨2点10分，备份日志文件a.log到/data/backup/log
- 备份开始和备份结束有提示
- 备份后的文件以时间命名为文件名，并打包成压缩包.tar.gz保存
- 在备份的时候检查是否有10天前备份过的日志文件，有的话就将它删除

## 示例：

```shell
#!/bin/bash
# 定时备份日志文件
# 定义备份的路径
BACKUP=/data/backup/log
# 获取当前时间作为文件名
DATETIME=`date +%Y_%m_%d_%H:%M:%S`

echo "=====开始备份====="
echo "=====备份的目标路径为$BACKUP====="
echo "=====文件名$DATETIME.tar.gz====="

#检查目标路径是否存在如果不存在就创建
[ ! -d $BACKUP ] && mkdir -p $BACKUP
DESTINFLOG=/root/a.log
# 检查目标文件是否存在
if [ ! -f $DESTINFLOG ]
then
        echo "$DESTINFLOG文件不存在！"
        exit
fi
		
# 开始备份
cp $DESTINFLOG $BACKUP/$DATETIME.log

#打包 (:会被识别为地址 需要--force-local)
cd $BACKUP
tar -zcvf $DATETIME.tar.gz $DATETIME.log --force-local

#删除临时文件
rm -rf $DATETIME.log

#检查是否有10天前的文件,全部删除
find $BACKUP -mtime +10 -name "*.tar.gz" -exec rm -rf {} \;

#结束
echo "=====备份结束====="
```

# 防火墙

## 查看防火墙情况

firewall-cmd --list -all

## 增加TCP端口

firewall-cmd --add-port=80/tcp --permanent

## 重新加载防火墙

firewall-cmd reload

## 常用命令

firewall-cmd [选项]

> -h, --help	显示帮助信息；
> -V, --version	显示版本信息. （这个选项不能与其他选项组合）；
> -q, --quiet	不打印状态消息；
> –state	显示firewalld的状态；
> –reload	不中断服务的重新加载；
> –complete-reload	中断所有连接的重新加载；
> –runtime-to-permanent	将当前防火墙的规则永久保存；
> –check-config	检查配置正确性；
> –get-log-denied	获取记录被拒绝的日志；
> –set-log-denied=	设置记录被拒绝的日志，只能为 ‘all’,‘unicast’,‘broadcast’,‘multicast’,‘off’ 其中的一个；

## 配置firewalld

```
firewall-cmd --version  # 查看版本
firewall-cmd --help     # 查看帮助

# 查看设置：
firewall-cmd --state  # 显示状态
firewall-cmd --get-active-zones  # 查看区域信息
firewall-cmd --get-zone-of-interface=eth0  # 查看指定接口所属区域
firewall-cmd --panic-on  # 拒绝所有包
firewall-cmd --panic-off  # 取消拒绝状态
firewall-cmd --query-panic  # 查看是否拒绝

firewall-cmd --reload # 更新防火墙规则
firewall-cmd --complete-reload
# 两者的区别就是第一个无需断开连接，就是firewalld特性之一动态添加规则，第二个需要断开连接，类似重启服务


# 将接口添加到区域，默认接口都在public
firewall-cmd --zone=public --add-interface=eth0
# 永久生效再加上 --permanent 然后reload防火墙

# 设置默认接口区域，立即生效无需重启
firewall-cmd --set-default-zone=public

# 查看所有打开的端口：
firewall-cmd --zone=dmz --list-ports

# 加入一个端口到区域：
firewall-cmd --zone=dmz --add-port=8080/tcp
# 若要永久生效方法同上

# 打开一个服务，类似于将端口可视化，服务需要在配置文件中添加，/etc/firewalld 目录下有services文件夹，这个不详细说了，详情参考文档
firewall-cmd --zone=work --add-service=smtp

# 移除服务
firewall-cmd --zone=work --remove-service=smtp

# 显示支持的区域列表
firewall-cmd --get-zones

# 设置为家庭区域
firewall-cmd --set-default-zone=home

# 查看当前区域
firewall-cmd --get-active-zones

# 设置当前区域的接口
firewall-cmd --get-zone-of-interface=enp03s

# 显示所有公共区域（public）
firewall-cmd --zone=public --list-all

# 临时修改网络接口（enp0s3）为内部区域（internal）
firewall-cmd --zone=internal --change-interface=enp03s

# 永久修改网络接口enp03s为内部区域（internal）
firewall-cmd --permanent --zone=internal --change-interface=enp03s

```

## 服务管理

```
# 显示服务列表  
Amanda, ftp, Samba和tftp等最重要的服务已经被FirewallD提供相应的服务，可以使用如下命令查看：

firewall-cmd --get-services

# 允许ssh服务通过
firewall-cmd --enable service=ssh

# 禁止SSH服务通过
firewall-cmd --disable service=ssh

# 打开TCP的8080端口
firewall-cmd --enable ports=8080/tcp

# 临时允许Samba服务通过600秒
firewall-cmd --enable service=samba --timeout=600

# 显示当前服务
firewall-cmd --list-services

# 添加HTTP服务到内部区域（internal）
firewall-cmd --permanent --zone=internal --add-service=http
firewall-cmd --reload     # 在不改变状态的条件下重新加载防火墙

```

## 端口管理

```
# 打开443/TCP端口
firewall-cmd --add-port=443/tcp

# 永久打开3690/TCP端口
firewall-cmd --permanent --add-port=3690/tcp

# 永久打开端口好像需要reload一下，临时打开好像不用，如果用了reload临时打开的端口就失效了
# 其它服务也可能是这样的，这个没有测试
firewall-cmd --reload

# 查看防火墙，添加的端口也可以看到
firewall-cmd --list-all

```

## 直接模式

```
# FirewallD包括一种直接模式，使用它可以完成一些工作，例如打开TCP协议的9999端口
firewall-cmd --direct -add-rule ipv4 filter INPUT 0 -p tcp --dport 9000 -j accept
firewall-cmd --reload

```

## 自定义服务管理

```
（末尾带有 [P only] 的话表示该选项除了与（--permanent）之外，不能与其他选项一同使用！）
--new-service=<服务名> 新建一个自定义服务 [P only]
--new-service-from-file=<文件名> [--name=<服务名>]
                      从文件中读取配置用以新建一个自定义服务 [P only]
--delete-service=<服务名>
                      删除一个已存在的服务 [P only]
--load-service-defaults=<服务名>
                      Load icmptype default settings [P only]
--info-service=<服务名>
                      显示该服务的相关信息
--path-service=<服务名>
                      显示该服务的文件的相关路径 [P only]
--service=<服务名> --set-description=<描述>
                      给该服务设置描述信息 [P only]
--service=<服务名> --get-description
                      显示该服务的描述信息 [P only]
--service=<服务名> --set-short=<描述>
                      给该服务设置一个简短的描述 [P only]
--service=<服务名> --get-short
                      显示该服务的简短描述 [P only]

--service=<服务名> --add-port=<端口号>[-<端口号>]/<protocol>
                      给该服务添加一个新的端口(端口段) [P only]

--service=<服务名> --remove-port=<端口号>[-<端口号>]/<protocol>
                      从该服务上移除一个端口(端口段) [P only]

--service=<服务名> --query-port=<端口号>[-<端口号>]/<protocol>
                      查询该服务是否添加了某个端口(端口段) [P only]

--service=<服务名> --get-ports
                      显示该服务添加的所有端口 [P only]

--service=<服务名> --add-protocol=<protocol>
                      为该服务添加一个协议 [P only]

--service=<服务名> --remove-protocol=<protocol>
                      从该服务上移除一个协议 [P only]

--service=<服务名> --query-protocol=<protocol>
                      查询该服务是否添加了某个协议 [P only]

--service=<服务名> --get-protocols
                      显示该服务添加的所有协议 [P only]

--service=<服务名> --add-source-port=<端口号>[-<端口号>]/<protocol>
                      添加新的源端口(端口段)到该服务 [P only]

--service=<服务名> --remove-source-port=<端口号>[-<端口号>]/<protocol>
                      从该服务中删除源端口(端口段) [P only]

--service=<服务名> --query-source-port=<端口号>[-<端口号>]/<protocol>
                      查询该服务是否添加了某个源端口(端口段) [P only]

--service=<服务名> --get-source-ports
                      显示该服务所有源端口 [P only]

--service=<服务名> --add-module=<module>
                      为该服务添加一个模块 [P only]
--service=<服务名> --remove-module=<module>
                      为该服务移除一个模块 [P only]
--service=<服务名> --query-module=<module>
                      查询该服务是否添加了某个模块 [P only]
--service=<服务名> --get-modules
                      显示该服务添加的所有模块 [P only]
--service=<服务名> --set-destination=<ipv>:<address>[/]
                      Set destination for ipv to address in service [P only]
--service=<服务名> --remove-destination=<ipv>
                      Disable destination for ipv i service [P only]
--service=<服务名> --query-destination=<ipv>:<address>[/]
                      Return whether destination ipv is set for service [P only]
--service=<服务名> --get-destinations
                      List destinations in service [P only]

```

## 控制端口/服务

>可以通过两种方式控制端口的开放，一种是指定端口号另一种是指定服务名。虽然开放 http 服务就是开放了 80 端口，但是还是不能通过端口号来关闭，也就是说通过指定服务名开放的就要通过指定服务名关闭；通过指定端口号开放的就要通过指定端口号关闭。还有一个要注意的就是指定端口的时候一定要指定是什么协议，tcp 还是 udp。知道这个之后以后就不用每次先关防火墙了，可以让防火墙真正的生效。

```
firewall-cmd --add-service=mysql        # 开放mysql端口
firewall-cmd --remove-service=http      # 阻止http端口
firewall-cmd --list-services            # 查看开放的服务
firewall-cmd --add-port=3306/tcp        # 开放通过tcp访问3306
firewall-cmd --remove-port=80tcp        # 阻止通过tcp访问3306
firewall-cmd --add-port=233/udp         # 开放通过udp访问233
firewall-cmd --list-ports               # 查看开放的端口

```

## 伪装IP

```
firewall-cmd --query-masquerade # 检查是否允许伪装IP
firewall-cmd --add-masquerade   # 允许防火墙伪装IP
firewall-cmd --remove-masquerade# 禁止防火墙伪装IP

```

## 端口转发

> 端口转发可以将指定地址访问指定的端口时，将流量转发至指定地址的指定端口。转发的目的如果不指定 ip 的话就默认为本机，如果指定了 ip 却没指定端口，则默认使用来源端口。 如果配置好端口转发之后不能用，可以检查下面两个问题：
>
> 比如我将 80 端口转发至 8080 端口，首先检查本地的 80 端口和目标的 8080 端口是否开放监听了
> 其次检查是否允许伪装 IP，没允许的话要开启伪装 IP

```
firewall-cmd --add-forward-port=port=80:proto=tcp:toport=8080   # 将80端口的流量转发至8080
firewall-cmd --add-forward-port=port=80:proto=tcp:toaddr=192.168.0.1 # 将80端口的流量转发至192.168.0.1
firewall-cmd --add-forward-port=port=80:proto=tcp:toaddr=192.168.0.1:toport=8080 # 将80端口的流量转发至192.168.0.1的8080端口

```

>- 1)当我们想把某个端口隐藏起来的时候，就可以在防火墙上阻止那个端口访问，然后再开一个不规则的端口，之后配置防火
>  墙的端口转发，将流量转发过去。
>- 2)端口转发还可以做流量分发，一个防火墙拖着好多台运行着不同服务的机器，然后用防火墙将不同端口的流量转发至不同机器。

# 开机启动服务

在linux下，有一个目录专门用以管理所有的服务的启动脚本的，即/etc/rc.d/init.d下。

这时候，我们来试一下使用service的命令进行启动，可以看到，这时候就可以通过service的方式进行启动了，真实运维中也是如此，我们只需要把服务的启动脚本文件放在此处，就可以以service的方式启动了

Reids示例：

每个服务都会配置一个init的脚本。例如redis在units下的，redis_init_script，就可以放在/etc/rc.d/init.d中

通过chkconfig 服务名 on就可以开机启动

**\# chkconfig:  2345 15 95** 其中2345是默认启动级别，级别有0-6共7个级别。15是启动优先级，95是停止优先级，优先级范围是0－100，数字越大，优先级越低。

# Jmap

jmap -dump:format=b,file=文件名 [pid]

```
jmap -dump:format=b,file=/usr/local/base/02.hprof 12942
```

-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/usr/local/base

# MemoryAnalyzer 

```
-vmargs -Xmx5g
```

# 信息查看

# 机器内核

/proc/version

uname -a

## 查看核心数

```SHELL
# 总核数 = 物理CPU个数 X 每颗物理CPU的核数 
# 总逻辑CPU数 = 物理CPU个数 X 每颗物理CPU的核数 X 超线程数

# 查看物理CPU个数
cat /proc/cpuinfo| grep "physical id"| sort| uniq| wc -l

# 查看每个物理CPU中core的个数(即核数)
cat /proc/cpuinfo| grep "cpu cores"| uniq

# 查看逻辑CPU的个数
cat /proc/cpuinfo| grep "processor"| wc -l
```



# ssh登陆

生成公私钥

ssh-keygen -t rsa -C "email" 

上传公钥到目标服务器

scp -p ~/.ssh/id_rsa.pub root@<remote_ip>:/root/.ssh/authorized_keys

## scp上传下载

```shell
-o PubkeyAuthentication=no
# 不实用pk下载
```

# 光标快捷键

```shell 
光标操作快捷键，光标快捷键

这些光标操作快捷键适用mac/linux终端和chrome控制台，这些快捷键都是emacs的快捷键，

常用的快捷键：

Ctrl + d        删除一个字符，相当于通常的Delete键(命令行若无所有字符，则相当于exit；处理多行标准输入时也表示eof)

Ctrl + h        退格删除一个字符，相当于通常的Backspace键

Ctrl + u        删除光标之前到行首的字符

Ctrl + k        删除光标之前到行尾的字符

Ctrl + c        取消当前行输入的命令，相当于Ctrl + Break

Ctrl + a        光标移动到行首(Ahead of line)，相当于通常的Home键

Ctrl + e        光标移动到行尾(End of line)

Ctrl + f        光标向前(Forward)移动一个字符位置

Ctrl + b        光标往回(Backward)移动一个字符位置

Ctrl + l        清屏，相当于执行clear命令

Ctrl + p        调出命令历史中的前一条(Previous)命令，相当于通常的上箭头

Ctrl + n        调出命令历史中的下一条(Next)命令，相当于通常的上箭头

Ctrl + r        显示：号提示，根据用户输入查找相关历史命令(reverse-i-search)

次常用快捷键：

Alt + f         光标向前(Forward)移动到下一个单词

Alt + b         光标往回(Backward)移动到前一个单词

Ctrl + w        删除从光标位置前到当前所处单词(Word)的开头

Alt + d         删除从光标位置到当前所处单词的末尾

Ctrl + y        粘贴最后一次被删除的单词


```

| COMMAND                  | ACTION                                                       |
| :----------------------- | :----------------------------------------------------------- |
| **Shortcuts**            |                                                              |
| Tab                      | Auto-complete file and folder names                          |
| Ctrl + A                 | Go to the beginning of the line you're currently typing on   |
| Ctrl + E                 | Go to the end of the line you're currently typing on         |
| Ctrl + U                 | Clear the line before the cursor                             |
| Ctrl + K                 | Clear the line after the cursor                              |
| Ctrl + W                 | Delete the word before the cursor                            |
| Ctrl + T                 | Swap the last two characters before the cursor               |
| Esc + T                  | Swap the last two words before the cursor                    |
| Ctrl + L                 | Clear the screen                                             |
| Ctrl + C                 | Kill whatever you're running                                 |
| Ctrl + D                 | Exit the current shell                                       |
| Option + →               | Move cursor one word forward                                 |
| Option + ←               | Move cursor one word backward                                |
| Ctrl + F                 | Move cursor one character forward                            |
| Ctrl + B                 | Move cursor one character backward                           |
| Ctrl + Y                 | Paste whatever was cut by the last command                   |
| Ctrl + Z                 | Puts whatever you're running into a suspended background process |
| Ctrl + _                 | Undo the last command                                        |
| Option + Shift + Cmd + C | Copy plain text                                              |
| Shift + Cmd + V          | Paste the selection                                          |
| exit                     | End a shell session                                          |

# 查看cpu架构

arch

## 设置网卡静态ip

```
BOOTPROTO=static

IPADDR=128.83.155.1
NETMASK=255.255.255.0
GATEWAY=128.83.155.250
DNS1=114.114.114.114


service network restart
```

## rpm和yum

https://www.cnblogs.com/LiuChunfu/p/8 052890.html

## aarch64 x86_64 arm64 amd64

https://www.jianshu.com/p/2753c45af9bf

aarch64==arm64 是arm架构的

x86_64=amd64 是x86架构的

# man详解

https://www.geek-share.com/detail/2728370984.html

# 设置hostname

