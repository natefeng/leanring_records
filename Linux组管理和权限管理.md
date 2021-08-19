# 10  Linux组管理和权限管理

## 10.1 Linux组基本介绍

在linux中的每一个用户必须属于一个组.每个文件有所有者，所在组,其他组的概念

 ## 10.2 文件/目录所有者

chgrp  :  改变档案所属群组

chown :  改变档案拥有者

chmod :  改变档案的权限, SUID, SGID, SBIT等等的特性

一般为文件的创建者,谁创建了该文件,就自然的成为该文件的所有者

 ### 10.2.1 查看文件的所有者

1) 指令: ls -ahl

 ### 修改文件的所有者

chown 用户名 文件名

## 10.3 组的创建

groupadd 组名

## 10.4 文件/目录 所在组

 当某个用户创建了一个文件后,该文件默认在所有者用户的的组

### 10.4.2 修改文件所在的组

 chgrp 组名 文件名 

## 10.6 改变用户所在组

在添加用户时候,可以指定将该用户添加到哪个组中,同样的用root的管理权限可以改变某个用户所在的组

usermod -g 组名 用户名 

usermod -d 目录名 用户名 设置用户登录的初始目录

![image-20200730224940283](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200730224940283.png)

## 10.7 权限的基本介绍

ls -l 中显示的内容如下

**-rwxrw-r-- 1 root root 1213 Feb 2 3月  18 09:39 abc**

第0位确定文件类型(d,-,l,c,b)

-代表普通文本 l 代表软链接,c代表字符

第1-3位确定该所有者拥有该文件的权限 r 读 w 写 x 执行 execute 执行 User

第4-6位确定该所属组拥有该文件的权限 Group

第7-10位确定其他组的用户拥有该文件的权限 Other

第一个root代表所有者

第二个root代表所属组 

两个都可以改变

1213代表大小

Feb 2  09:39 代表该文件最后一次的修改时间 

abc 代表该文件的名字



![image-20200730225416461](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200730225416461.png)

## 10.8 权限详解

### rwx作用到文件

1) 【r】代表可以读取,查看 read

2) 【w】代表可以修改 但是不可以删除 如果想要删除必须保证对文件所在的目录拥有写的权限才可以删除

3)  【x】代表可以被执行 execute 

### rwx作用到目录

1)【r】代表可以读取 ls 可以查看目标内容

2) 【w】代表可以修改,目录内部可以创建+修改+重命名子目录

3) 【x】代表可可以进入该目录

## 10.10 修改权限-chmod

 ### 10.10.1 基本说明和各种变更权限方式

通过chmod 指令,可以修改文件或者目录的权限

 【u】所有者 【g】所属组 【o】其他组的用户 【o】所有人

1) chmod u=rwx,g=rx,o=x 文件或者目录名

2) chmod o+w 文件目录名

3) chmod o-w 文件目录名

### 10.10.2 第二种数组变更权限的方式

r = 4 ,w = 2,x = 1 rwx = 4+2+1 = 7

 ##  

## 10.11 修改文件所有者 chowm

chown newowner file 改变文件的所有者

chown newowner:newgruop file 该表文件的所有者和所有组

-R 如果该变目录加此

## 10.12 修改文件所在组 -chgrp

chgrp newgroup file 

![image-20200730230800912](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200730230800912.png)

# 第十一章 实操篇 任务调度

 任务调度: 是指系统在某个时间执行的特定的命令或程序

 任务调度分类: 1.系统工作: 有些重要的工作必须周而复始的进行,如按时扫描病毒

![image-20200731125717985](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200731125717985.png)

crontab [选项]  crontab -e 编辑定时任务

crontab [选项]  crontab -r 停止定时任务

crontab [选项]  crontab -l 查询当前用户的crontab任务

设置任务调度文件: /ect/crontab 

在里面编写 或者编写shell脚本代码

![image-20200731170630694](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200731170630694.png)

  每分钟把ls -l /ect 下的内容 覆盖到tmp/to.txt 文本下

![image-20200731190340109](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200731190340109.png)

![image-20200731190346493](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200731190346493.png)

![image-20200731190358824](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200731190358824.png) 





## 11.5 案例 每隔1分钟就将当前的日期信息,追加到 /tmp/mydate

编写一个脚本文件 /home/mytask1.sh

在mytask1.sh编写

 date ->> /tmp/mydate

然后 crontab -e 里面编写 */1 * * * *  /home/mytask1.sh

必须保证mytask1.sh文件是可执行文件

将Mysql 数据库testdb,备份到文件中mydb.bak.

/usr/local/mysql/bin/mysqldump -u root -p123456 testdb > /tmp/mydb.bak

# 12 Linux 磁盘分区,挂载 

### 分区的方式



1) mbr 分区

1.最多支持四个主分区

2.系统只能安装在主分区

3.扩展分区要占一个主分区

4.MBR最大支持2TB,但是拥有最好的兼容性

2)gtp分区:

1）支持无线多个主分区(但是操作系统可能限制,比如window下最多128个分区)

2) 最大支持18EB （1EB = 1024 PB,1PB = 1024 TB）

3）windows7 64位以后支持gtp

### 12.1.2 windows下的磁盘分区

![image-20200731215545957](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200731215545957.png)

## 12.2 Linux分区

### 12.2.1 原理介绍

1)Linux来说无论有几个分区,分给哪一目录使用,它归根结底只有一个根目录,一个独立且唯一的文件结构,Linux中每个分区都是用来组成整个文件系统的一部分

2) Linux采用了一种载入的处理方法,它的整个文件系统中包含了一整套的文件和目录,且将一个分区和一个目录联系起来,这时要载入的一个分区将使它的存储空间在一个目录下获得

![image-20200731220527354](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200731220527354.png)



### 12.2.2 使用lsblk 指令查看当前系统的分区情况

![image-20200731220834210](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200731220834210.png)

挂载一块2G的硬盘到/home/newdisk目录下

首先创建一个newdisk目录

然后在虚拟机中新增一块2G的硬盘

然后重启虚拟机 reboot

然后开机输入 lsblk 查看当前分区情况 发现已经有一个硬盘但是未分区

输入fdisk /dev/sdb(磁盘名)    该代码是使硬盘分区

分区后需要格式化

mkfs -t ext4 /dev/sdb1  将该分区格式化为ext4文件系统

挂载 mount /dev/sdb1 /home/newdisk 将sdb1这个硬盘挂载到该目录下也就是产生联系

上面的挂载方式重启就会失效

如果需要永久挂载

vim /ect/fstab

yyp 复制一行

/dev/sdb1     /home/newdisk ext4 defaults 0 0

取消联系代码

![image-20200731221526728](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200731221526728.png)

# 13 磁盘实用指令

查看当前磁盘详情

df -lh human 人类 和df -l 

查询系统整体磁盘使用情况

![image-20200801221247205](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801221247205.png)

基本语法 

​    du -h /目录

查看指定目录的磁盘占用情况,默认为当前目录

 -s  指定目录占用大小汇总

-h 带计量单位

-a 含文件

--max-deph=1 子目录深度

-c 列出明细的同时,增加汇总值

应用实例

 查询/opt目录下的占用情况,深度为1(也就是只查一层)

![](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801221812286.png)

## 13.2 磁盘情况-工作实用指令

1）统计/home文件夹下文件的个数

 ls -l  /home | grep "^-" | wc -l

2) 统计/home文件夹下目录的个数

 ls -l  /home | grep "^d" | wc -l

3) 统计/home文件夹文件的个数,包括子文件夹

ls -lR /home | grep "^-" | wc -l

4）树状显示目录结构

yum install tree  安装tree命令

![image-20200801222356630](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801222356630.png)



# 14  实操篇 网络配置

目前我们的网路配置采用的是Nat

![image-20200801222642414](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801222642414.png)

 ## 14.2 查看网络IP和网关

![image-20200801222733989](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801222733989.png)

![image-20200801222746181](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801222746181.png)

![image-20200801222756487](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801222756487.png)

## 14.3 Linux 网络环境配置的两种方式

### 14.3.1 自动获取

![image-20200801222849799](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801222849799.png)

缺点: linux 启动后会自动获取IP,缺点是每次自动获取的ip地址可能不一样。这个不适用于做服务器,因为我们的服务器的 ip 需要时固定的.

### 14.3.2 第二种方式（静态指定IP）

 说明

直接修改配置文件来指定ip,并可以连接到外网(程序员推荐) ,编辑

vim /ect/sysconfig/network-scripts/ifcfg-eth0

![image-20200801223330296](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801223330296.png)

![image-20200801223344932](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801223344932.png)

将配置文件改成本地和网卡一致的IP段

修改后,一定要重启服务

1) service network restart

2) reboot 重启系统

# 15 进程管理

## 15.1 进程的基本介绍

要想通过ssh链接远程服务器，对方服务器需要安装ssh服务器端程序，并开启对应服务。
CentOS中默认使用的是openssh-server，进程名称为sshd，默认监听的网络端口是22。

Bios自检

1) 在Linux中,每个执行的程序(代码)都称为一个进程.每一个进程都有一个ID号

2) 每一个进程,都会对应一个父进程，而且这个父进程可以复制多个子进程。例如www服务器

3) 每个进程都可能以两种方式存在的。前台与后台，所谓前台进程就是用户目前的屏幕上可以进行操作的。后台进程则是实际在操作,但是由于屏幕上无法看到的进程,通常使用后台方式执行。

4) 一般系统的服务都是以后台进程的方式存在,而且都会常驻在系统中。直到关机才结束。

## 15.2 显示系统执行的进程

### 15.2.1 说明:

 查看进程使用的指令是 ps ,一般来说使用的参数 是 ps -aux

![image-20200801224815488](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801224815488.png)

![image-20200801224824303](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801224824303.png)

IPD进程ID号 USER 用户名 VSZ 虚拟内存 TTY使用的终端

STAT 进程的状态  S ：睡眠  -s:表示该进程是会话的先导进程.N-表示该进程比普通进程更低的优先级

R- 正在运行 D- 短期等待 Z- 僵死进程，T-被跟踪或者被停止等等

STARTED : 进程的启动实际

TIME: CPU时间,进程使用的CPU总时间

COMMADN：启动进程所用的命令和参数 

1) ps -aux | grep xxx 比如sshd 查看sshd服务的

### 15.2.3 应用实例

要求 : 以全格式展示当前所有的进程,查看进程的父进程

ps -ef | more 

![image-20200801225416784](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801225416784.png)

![image-20200801225433803](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801225433803.png)

思考题 , 如果我们希望查看 sshd 进程的父进程号是多少，应该怎样查询？

ps -ef | grep sshd

![image-20200801225651653](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801225651653.png)

## 15.3 终止进程 kill 和 killall

### 15.3.1 介绍

若是某个进程执行一半需要停止时，或是已消耗了很大的系统资源时，此使可以考虑停止该进程使用kill命令来完成此项任务

### 15.3.2 基本语法

kill[选项]进程号(功能描述：通过进程号杀死进程)

killall 进程名称 （功能描述：通过进程名称杀死进程,也支持通配符，这在系统因负载过大而变得很慢时特别有用）

### 15.3.3 常用选项：

-9 表示强迫进程立刻停止

### 15.3.4 最佳实践

案例1：踢掉某个非法登录用户

ps -aux | grep sshd

![image-20200801230131291](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801230131291.png)

![image-20200801230147105](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801230147105.png)

案例三 ：终止多个getit编辑器

 killall gedit

案例四 ： 强制杀掉一个终端

首先查看终端

ps -aux | grep bash

然后利用kill -9 id号 强制杀掉！！

## 15.4 查看进程数 pstree

### 15.4.1 基本语法

pstree[选项]，可以更加直观的来看进程信息

-p : 显示进程的IPD

-u ：显示进程的所属用户

![image-20200801230832761](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801230832761.png)

## 15.5 服务(Service)管理

### 15.5.1 介绍:

服务(service)本质就是进程,但是是运行在后台的，通常都会监听某个端口,等待其他程序的请求,比如(mysql,sshd 防火墙等),因此我们又称为守护进程.是Linux非常重要的知识点【原理图】

![image-20200801231027316](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801231027316.png)

### 15.5.2 service 管理指令

service 服务名【start | stop | restart | reload | status】

在Centos7.0后 不再使用service,而是 systemctl

### 15.5.3 使用案例

1) 查看当前防火墙的状况，关闭防火墙和重启防火墙

service iptables stop

service iptables restart

![image-20200801231320658](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801231320658.png)

### 15.5.4 细节讨论

1）关闭或者启用防火墙后，立即生效【telnet 测试 某个端口即可】

telnet ip 端口号

2）这种方式只是临时生效，如果重启系统后,会回归原来以前对服务的设置

如果希望永久生效,要使用chkconfig指令

### 15.5.5 查看服务名

方式1：setup--> 系统服务 

![image-20200801231527398](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801231527398.png)

方式2：/etc/init.d/服务名称

![image-20200801231612454](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801231612454.png)



![image-20200801231618143](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801231618143.png)

开机的流程说明

首先开机 然后BIOS自检 /boot引导  开启init进程然后依据inittab文件来读取运行等级  内核被加载后,第一个运行的程序时/sbin/inint 该文件会读取/ect/inittab文件 并且读取到运行Linux对应的运行级别

![image-20200801231937603](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801231937603.png)

### 15.5.6 chkconfig指令

介绍

通过chkconfig 命令可以给每个服务的各个运行级别设置自启动/关闭

基本语法

1）查看服务 

  chkconfig --list

![image-20200801232136945](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801232136945.png)

chkconfig --list | grep sshd

![image-20200801232203323](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801232203323.png)

![image-20200801232207895](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801232207895.png)

3）给某个服务的某个运行级别设置对应的自启动/关闭

chkconfig --level 5 服务名 on/off

![image-20200801232326947](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801232326947.png)

案例一 查看当前服务的各个运行级别的运行状态

chkconfig --list

案例二 查看sshd服务的运行状态

chkconfig --list | grep sshd

chkconfig sshd --list

案例三 将sshd 服务在运行级别5下设置为不自动启动,看看有什么效果？

chkconfig --level 5 sshd off

案例四：当运行级别为5时，关闭防火墙

chkconfig --level 5 iptables off

![image-20200801232625401](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801232625401.png)

## 15.6 动态监控进程

### 15.6.1 介绍

top与 ps 命令很相似。它们都用显示正在执行的进程。top 和 ps 最大的不同之处，在于top执行一段时间可以更新正在运行的进程。

14.6.2 基本语法

top[选项]

-d 设置top命令每隔几秒更新 默认3秒 可以更改

-i 使top不显示任何闲置或者僵尸进程

![image-20200801233031344](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801233031344.png)

-p 通过指定监控进程ID来查询某个进程的状态

![image-20200801233054978](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801233054978.png)

### 15.6.2 应用案例

监视特定用户

top

![image-20200801233131911](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801233131911.png)

![image-20200801233142085](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801233142085.png)

![image-20200801233151296](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801233151296.png)

### 15.6.3 查看系统网络情况 netstat(重要)

基本语法

 netstart[选项]

   netstat -anp

选项说明

-an 按一定顺序排列输出

-p 显示哪个进程在调用

netstat -anp

![image-20200801233304391](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200801233304391.png)

0.0.0.0 代表正在监听

# RPM和YUM

## 16.1 rpm包的管理

### 16.1.1 介绍

一种用于互联网下载包的打包及安装工具，它包含在某些Linux发行版中，它生成具有.RPM扩展名的文件。RPM是RedHat package manager(RedHat软件包管理工具)的缩写

### 16.1.2 rpm包的简单查询指令

查询已安装的rpm列表 rpm -qa|grep xx

查询看一下有没有安装火狐 rpm -qa | grep firefox

![image-20200806123340664](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806123340664.png)



### 16.1.3 rpm包的其他查询指令

rpm -qa:查询所有的rpm软件包

rpm -qa|more[分页显示]

rpm -qa|grep X[rpm -qa | grep firefox]

rpm -q 软件包名 ：查询软件包是否安装

rpm -qi 软件包名：查询软件包信息

![image-20200806123735596](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806123735596.png)

rpm -ql firefox 查询rpm包的文件安装到哪里去了

### 16.1.4  卸载rpm包

基本语法

rpm -e RPM包的名称

![image-20200806123948896](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806123948896.png)

rpm -e --nodeps foo

### 16.1.5 安装rpm包

基本语法

 rpm ivh

参数说明 i = install v = verbose 提示 h = hash 仔细 进度条

案例演示

![image-20200806124157622](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806124157622.png)

## 16.2 YUM

### 16.2.1 介绍

Yum是一个Shell前端软件包管理器。基于RPM包管理，能够从指定的服务器自动下载RPM包并且安装，可以自动处理依赖关系，并且一次安装所有的依赖包。使用yum的提前必须有网络

![image-20200806124347796](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806124347796.png)

### 16.2.2 yum的基本指令

查询yum服务器是否有需要安装的软件

yum list|grep xx 软件名

yum install xxx 下载安装

### 16.2.3应用案例

![image-20200806124604587](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806124604587.png)

![image-20200806124611938](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806124611938.png)

# JAVAEE定制篇 搭建JAVAEE环境

JAVA_HOME=java的家

PATH=java的家里面的bin目录$PATH $PATH指令是指加上原有的PATH一些的命令

export JAVA_HOME PATH 把这两个命令当成环境变量输出

开放8080端口

vim /ect/sysconfig/iptables

# 17大数据定制篇Shell编程

## 17.1 为什么要学习shell编程

1）Linux运维工程师在进行服务器集群管理时，需要编写shell程序来进行服务器管理

2）对于JAVAEE和PATHON程序员来说，工作的需要，你的老大会要求你编写一些shell脚本进行程序或者是服务器的维护，比如编写一个定时备份数据库的脚本

### 17.2 shell是什么

![image-20200806125223838](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806125223838.png)

shell是一个命令解析器，它为用户提供了一个向Linux内核发送请求以便运行程序的界面系统级程序

## 17.3 shell编程快速入门-shell脚本的执行方式

### 17.3.1	 脚本格式要求

1）脚本以#!/bin/bash开头

2)脚本需要有一个可执行的权限

![image-20200806125618819](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806125618819.png)

## 17.4 shell的变量

 ### 17.4.1 Shell的变量的介绍

1）Linux Shell中的变量分为：系统变量和用户自定义变量

2）系统变量：$HOME,$PWD,$SHELL,$USER 等待

![image-20200806125756674](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806125756674.png)

3)显示当前shell中的所有变量 ：set

### 17.4.2 shell变量的定义

基本语法

定义变量 变量名=值

撤销变量 unset 变量名

声明静态变量 readonly 变量名 注意 不能unset

### 17.4.3 定义变量的规则

1）变量名称可以由字母，数字和下划线组成，但是不能以数字开头

2）等号两侧不能有空格

3）变量名称一般习惯为大写

### 17.4.4 将命令的返回值赋给变量（重点）

A=`ls -la` 反引号 运行里面的命令，并且把结果返回给A

A=$(ls -la) 等价于上面的

![image-20200806130152276](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806130152276.png)

 ## 17.5 设置环境变量

### 17.5.1 基本语法

1）export 变量名=变量值（功能描述 将shell变量输出为环境变量）

2)   source 配置文件 （功能描述：让修改的配置信息立即生效）

3)   echo $ 变量名 （输入环境变量的值）![image-20200806130404101](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806130404101.png)

### 17.5.2 快速入门

在/ect/profile编写TOMCAT_Home

![image-20200806130442253](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806130442253.png)

![image-20200806130455978](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806130455978.png)

## 17.6 位置参数变量

### 17.6.1 介绍

当我们执行一个shell脚本时，如果希望获取到命令行的参数信息，就可以使用到位置参数的变量 如何https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/myshell.sh 100 200 这个就是执行shell的命令行 

### 17.6.2 基本语法

$n(功能描述：n为数字 $0代表命令本身，$1-$9 代表第一到第九个参数，十以上的参数需要用大括号$(10))

$*(功能描述：把参数看作一个整体)

$@(也代表所有参数，不过把每个参数区分对待)

$#(参数的总个数)

## 17.7 预定义变量

### 17.7.1 基本介绍

就是shell设计者事先已经定义好的变量，可以直接在shell脚本中使用

$$(功能描述：当前进程的进程号)

$!(功能描述：后台运行的最后一个进程号)

$?(功能描述：最后一次执行的命令的返回状态 如果返回为0则代表正确，否则错误)

![image-20200806131133479](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806131133479.png)

https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/myShell.sh & 的意思是以执行这个shell脚本

## 17.8 运算符

### 17.8.1 基本介绍

学习如何在shell中进行各种运算操作

### 17.8.2 基本语法

1）"$((运算式))"或者 $[运算式]

2) expr m + n

注意 expr运算符间要有空格

3）expr /*,/,% 乘 除 取余

![image-20200806131500899](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806131500899.png)

#!/bin/bash

RESULT2=$[(2+3)*4]

echo $RESULT2

求命令行两个参数的和

$[$1+$2]

## 17.9 条件判断

### 17.9.1 基本语法

condition 条件

[ condition ](注意condition前后要有空格)

非空返回true，可使用$?验证（0为true，>1为false）

![image-20200806131822410](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806131822410.png)

### 17.9.2 常用判断条件

1)两个整数的比较

== 字符串比较 -ls 小于 -le小于等于 eq 等于 gt 大于 ge 大于等于 -ne 不等于

2）按照文件权限进行判断

-r 有读的权限 [-r 文件名]

-x 有执行的权限

-w 有写的权限

3)按照文件类型进行判断

-f 文件存在并且是一个常规的文件

-e 文件存在

-d 文件存在并且是一个目录

### 17.9.3 应用实例

案例1：“ok” 是否等于“ok”

判断语句

if  [ "ok"=="ok" ]

then

​      echo "equla"

fi

### 17.10 流程控制

## 17.10.1 if语句

基本语法

if [ 条件判断式 ];then

程序

fi

或者

if [ 条件判断式 ]

then

 程序

​      elif [ 条件判断式 ]

then

 程序

fi

![image-20200806132325612](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806132325612.png)

### 17.10.2 case语句

case $变量名 in

 "值 1")

如果变量的值等于1，则执行程序1

;;

"值2")

程序2

;;

*) 如果变量的值都不是以上的值，则执行此程序

;;

esac

![、](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806132531335.png)

### 17.10.3 for 循环

基本语法

for 变量 in 值1 值2 值3等等

do

程序

done

案例1：打印命令行输入的参数【会使用到$*$@】

![image-20200806132900231](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806132900231.png)

以上一句的意思是

把$*赋值给i

然后输出

基本语法2

for((初始值;循环控制条件;变量变化))

do

程序

done

![image-20200806133017684](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806133017684.png)

### 17.10.4 while循环

基本语法1

while[ 条件判断式 ]

do

​    程序

done

![image-20200806133110707](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806133110707.png) 

## 17.11 read读取控制台输入

### 17.11.1基本语法

read(选项)(参数)

选项:

​         -p:指定读取值的提示符；

​         -t：指定读取值时等待的时间(秒)，如果没有在指定的时间输入，就不等待

![image-20200806133319455](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806133319455.png)

## 17.12函数

### 17.12.1函数介绍

shell编程和其他编程语言一样,有系统函数，也可以自定义函数。在系统函数中，我们这里就介绍两个

### 17.2.2 系统函数

basename 基本语法

功能：返回完整路径最后/的语法，常用于获取文件名

bashname [pathname] [ suffix ]

bashname [string] [ suffix ]

选项：

suffix为后缀，如果后缀指定了,basename会将pathname或者string的suffix去掉

dirname 基本语法

功能：返回完整路径最后 / 的前面部分 常用于返回路径部分

dirname 文件绝对路径(功能描述：从给定的包含绝对路径的文件名去除文件名)

案例1：请返回/home/aaa/test.txt的test.txt部分

basename /home/aaa/test.txt

basename /home/aaa/test.txt .txt

![image-20200806133936164](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806133936164.png)

### 17.12.3 自定义函数

Action 行动

基本语法

 [ function ] funname[()]{

​     Action;

​      [return int;]

} 

![image-20200806135640393](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20200806135640393.png)