---
title: Linux基础系统优化
date: '2021-08-19T00:00:00+08:00'
tags:
- linux
showToc: true
categories:
- cloud
---



## Linux基础系统优化

Linux的网络功能相当强悍，一时之间我们无法了解所有的网络命令，在配置服务器基础环境时，先了解下网络参数设定命令。

*   ifconfig　　查询、设置网卡和ip等参数
*   ifup,ifdown 脚本命令，更简单的方式启动关闭网络
*   ip　　符合指令，直接修改上述功能

在我们刚装好linux的时候，需要用xshell进行远程连接，那就得获取ip地址，有时候网卡默认是没启动的，Linux也就拿不到ip地址，因此我们得手动启动网卡

1.  编辑网卡配置文件vim /etc/sysconfig/network-scripts/ifcfg-eth0
2.  修改配置参数ONBOOT=yes

### 网卡配置文件详解

1.  网络配置文件：/etc/sysconfig/network
2.  网络接口配置文件：etc/sysconfig/network-scripts/ifcfg-INTERFACE_NAME
3.  DEVICE=: 关联的设备名称，要与文件名的后半部“INTERFACE_NAME”保持一致;
4.  BOOTPROTO={static|none|dhcp|bootp}:引导协议；要使用静态地址，使用static或none；dhcp表示使用DHCP服务器获取地址；
5.  IPADDR=IP地址
6.  NETMASK=：子网掩码
7.  GATEWAY=：设定默认网关；
8.  ONBOOT=：开机时是否自动激活此网络接口；
9.  HWADDR=： 硬件地址，要与硬件中的地址保持一致；可省；
10.  USERCTL={yes|no}: 是否允许普通用户控制此接口；
11.  PEERDNS={yes|no}: 是否在BOOTPROTO为dhcp时接受由DHCP服务器指定的DNS地址；

### ifup,ifdown命令

1.  启动/关闭一块网卡
2.  ifup eth0
3.  ifdown eth0

### ifconfig命令

1.  ifconfig 查看网卡的ip地址

![](/images/f8a9aaf4-c0e7-4b0e-9ac3-30d4a6c03ef5.png)

1.  接输入ifconfig会列出已经启动的网卡，也可以输入ifconfig eth0单独显示eth0的信息
2.  选项解释是：
3.  th0 网卡的代号
4.  o 回环地址loopback
5.  net IPv4的Ip地址
6.  etmask 子网掩码
7.  roadcast 广播地址
8.  X/TX 流量发/收情况 tx是发送（transport），rx是接收(receive)
9.  ackets 数据包数
10.  rrors 数据包错误数
11.  ropped 数据包有问题被丢弃的数量
12.  ollisions 数据包碰撞情况，数值太多代表网络状况差

### ifup,ifdown命令

1.  ifup和ifdown是直接连接到/etc/sysconfig/network-scripts目录下搜索对应的网卡文件，例如ifcfg-eth0然后加以设置

### ip命令

1.  ip是一个命令，不是TCP/IP那个ip，这个ip命令是结合了ifconfig和route两个命令的功能。
2.  ip addr show #查看ip信息

了解了如何查看网卡信息，接下来查看系统信息。

你的系统是什么版本？

1.  查看系统版本信息cat /etc/redhat-release
2.  查看内核版本号uname -r
3.  查看系统多少位uname -m
4.  查看内核所有信息uname -a

## 用户管理与文件权限篇

### 创建普通用户

1.  添加用户useradd zhaohaiyu
2.  设置密码passwd zhaohaiyuroot用户可以修改其他所有人的密码，且不需要验证

### 切换用户

*   su - username

*   先看下当前用户（我是谁）whoami

*   退出用户登录logoutctrl + d

*   一般情况下，在生产环境避免直接用root用户，除非有特殊系统维护需求，使用完立刻退回普通用户

*   非交互式设置密码(echo "redhat"|passwd --stdin oldboy && history -c)

```
Tip:
1.超级用户root切换普通用户无需密码,例如“群主”想踢谁就踢谁
2.普通用户切换root，需要输入密码
3.普通用户权限较小，只能基本查看信息
4.$符号是普通用户命令提示符，#是超级管理员的提示符
root是当前用户，oldboyedu是主机名，~代表当前路径，也是家目录
```

groupadd命令

group命令用于创建用户组，为了更加高效的指派系统中各个用户的权限，groupadd it_dep

### userdel删除用户

*   -f 强制删除用户
*   -r 同事删除用户以及家目录
*   userdel -r zhaohaiyu

## 文件与目录权限

Linux权限的目的是（保护账户的资料）

Linux权限主要依据三种身份来决定：

*   user/owner 文件使用者,文件属于哪个用户
*   group 属组,文件属于哪个组
*   others 既不是user，也不再group，就是other，其他人

### 什么是权限

在Linux中，每个文件都有所属的所有者，和所有组，并且规定了文件的所有者，所有组以及其他人对文件的，可读，可写，可执行等权限。对于目录的权限来说，可读是读取目录文件列表，可写是表示在目录内新增，修改，删除文件。可执行表示可以进入目录

### Linux权限的观察

使用一条命令查看权限ls -l /var/log/mysqld.log

![](/images/44d5a029-5692-45c8-8c46-df07064b564a.png)

1.  权限，第一个字母为文件类型，后续9个字母，每3个一组，是三种身份的权限
2.  文件链接数
3.  文件拥有者-属主
4.  文件拥有组-属组
5.  文件大小
6.  最后一次被修改的时间日期
7.  文件名

先来分析一下文件的类型

```
-    一般文件
d    文件夹
l    软连接（快捷方式）
b    块设备，存储媒体文件为主
c    代表键盘,鼠标等设备
```

### 文件权限

```
r    read可读，可以用cat等命令查看
w    write写入，可以编辑或者删除这个文件
x    executable    可以执行
```

## 目录权限

```
r    可以对此目录执行ls列出所有文件
w    可以在这个目录创建文件
x    可以cd进入这个目录，或者查看详细信息
```

权限与数字转化

![](/images/0bbf82e6-b6bc-4c00-85c4-7122c61495e1.png)

## 软连接

软连接也叫做符号链接，类似于windows的快捷方式。

常用于安装软件的快捷方式配置，如mysql，nginx等ln -s 目标文件 软连接名

1.  存在文件/tmp/test.txt-rw-r--r-- 1 root root 10 10月 15 21:23 test.txt
2.  在/home目录中建立软连接，指向/tmp/test.txt文件=ln -s /tmp/test.txt my_test
3.  查看软连接信息
    lrwxrwxrwx 1 root root 13 10月 15 21:35 my_test -> /tmp/test.txt4.通过软连接查看文件cat my_testmy_test只是/tmp/test.txt的一个别名，因此删除my_test不会影响/tmp/test.txt，但是删除了本尊，快捷方式就无意义不存在了

## 命令

## tar解压命令

linux的文件打包工具最出名的是tar。

![](/images/c62e4613-cf4c-40c6-b48e-3429a816f541.png)

tar 命令：用来压缩和解压文件。tar本身不具有压缩功能。他是调用压缩功能实现的

语法

```
tar(选项)(参数)
-A或--catenate：新增文件到以存在的备份文件；
-B：设置区块大小；
-c或--create：建立新的备份文件；
-C ：这个选项用在解压缩，若要在特定目录解压缩，可以使用这个选项。
-d：记录文件的差别；
-x或--extract或--get：从备份文件中还原文件；
-t或--list：列出备份文件的内容；
-z或--gzip或--ungzip：通过gzip指令处理备份文件；
-Z或--compress或--uncompress：通过compress指令处理备份文件；
-f或--file=：指定备份文件；
-v或--verbose：显示指令执行过程；
-r：添加文件到已经压缩的文件；
-u：添加改变了和现有的文件到已经存在的压缩文件；
-j：支持bzip2解压文件；
-v：显示操作过程；
-l：文件系统边界设置；
-k：保留原有文件不覆盖；
-m：保留文件不被覆盖；
-w：确认压缩文件的正确性；
-p或--same-permissions：用原来的文件权限还原文件；
-P或--absolute-names：文件名使用绝对名称，不移除文件名称前的“/”号；
-N  或 --newer=：只将较指定日期更新的文件保存到备份文件里；
--exclude=：排除符合范本样式的文件。
```

## gzip命令

*   gzip用来压缩文件，是个使用广泛的压缩程序，被压缩的以".gz"扩展名
*   gzip可以压缩较大的文件，以60%~70%压缩率来节省磁盘空间

语法

```
-d或--decompress或----uncompress：解开压缩文件；
-f或——force：强行压缩文件。
-h或——help：在线帮助；
-l或——list：列出压缩文件的相关信息；
-L或——license：显示版本与版权信息；
-r或——recursive：递归处理，将指定目录下的所有文件及子目录一并处理；
-v或——verbose：显示指令执行过程；
```

## netstat命令

netstat命令用来打印Linux中网络系统的状态信息，可让你得知整个Linux系统的网络情况。

语法【选项】

```
netstat [选项]
-t或--tcp：显示TCP传输协议的连线状况；
-u或--udp：显示UDP传输协议的连线状况；
-n或--numeric：直接使用ip地址，而不通过域名服务器；
-l或--listening：显示监控中的服务器的Socket；
-p或--programs：显示正在使用Socket的程序识别码和程序名称；-a或--all：显示所有连线中的Socket；
```

## ps命令

ps 命令用于查看系统中的进程状态，格式为“ps [参数]”。

```
ps　　命令常用参数
-a     显示所有进程
-u     用户以及其他详细信息
-x    显示没有控制终端的进程
```

## Kill命令

kill命令用来删除执行中的程序或工作。kill可将指定的信息送至程序。

选项

```
-a：当处理当前进程时，不限制命令名和进程号的对应关系；
-l ：若不加选项，则-l参数会列出全部的信息名称；
-p：指定kill 命令只打印相关进程的进程号，而不发送任何信号；
-s ：指定要送出的信息；
-u：指定用户。
```

只有第9种信号(SIGKILL)才可以无条件终止进程，其他信号进程都有权利忽略，**下面是常用的信号：**

```
HUP     1    终端断线
INT     2    中断（同 Ctrl + C）
QUIT    3    退出（同 Ctrl + \）
TERM   15    终止
KILL    9    强制终止
CONT   18    继续（与STOP相反， fg/bg命令）
STOP   19    暂停（同 Ctrl + Z）
```

## killall命令

通常来讲，复杂软件的服务程序会有多个进程协同为用户提供服务，如果逐个去结束这 些进程会比较麻烦，此时可以使用 killall 命令来批量结束某个服务程序带有的全部进程。
例如nginx启动后有2个进程
killall nginx

## SELinux功能

SELinux(Security-Enhanced Linux) 是美国国家安全局（NSA）对于强制访问控制的实现，这个功能管理员又爱又恨，大多数生产环境也是关闭的做法，安全手段使用其他方法。

大多数ssh连接不上虚拟机，都是因为防火墙和selinux阻挡了

永久关闭方式：

```
1.修改配置文件，永久生效关闭selinux
cp /etc/selinux/config /etc/selinux/config.bak #修改前备份
2.修改方式可以vim编辑,找到
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
3.用sed替换
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
4.检查状态
grep "SELINUX=disabled" /etc/selinux/config
#出现结果即表示修改成功
```

临时关闭selinux(命令行修改，重启失效)：

```
getenforce #获取selinux状态
#修改selinux状态
setenforce 
usage:  setenforce [ Enforcing | Permissive | 1 | 0 ]
数字0 表示permissive，给出警告，不会阻止，等同disabled
数字1表示enforcing，表示开启
```

Tip:

```
修改selinux配置后，想要生效还得重启系统，技巧就是（修改配置文件+命令行修改，达到立即生效）
生产环境的服务器是禁止随意重启的！！！！
```

## iptables防火墙

在学习阶段，关闭防火墙可以更方便的学习，在企业环境中，一般只有配置外网ip的linux服务器才会开启防火墙，但是对于高并发流量的业务服务器仍然是不能开启的，会有很大性能损失，因此需要更nb的硬件防火墙。

关闭防火墙具体操作如下：

1.  systemctl status firewalld查看防火墙状态
2.  systemctl stop firewalld关闭防火墙
3.  systemctl disable firewalld关闭防火墙开机启动systemctl is-enabled firewalld.service#检查防火墙是否启动

## df命令

**df命令**用于显示磁盘分区上的可使用的磁盘空间。默认显示单位为KB。可以利用该命令来获取硬盘被占用了多少空间，目前还剩下多少空间等信息。

语法

```
df(选项)(参数)
-h或--human-readable：以可读性较高的方式来显示信息；
-k或--kilobytes：指定区块大小为1024字节；
-T或--print-type：显示文件系统的类型；
--help：显示帮助；
--version：显示版本信息。
```

## tree命令

1.  tree命令以树状图列出目录的内容。
2.  -a：显示所有文件和目录；
3.  -A：使用ASNI绘图字符显示树状图而非以ASCII字符组合；
4.  -C：在文件和目录清单加上色彩，便于区分各种类型；
5.  -d：先是目录名称而非内容；
6.  -D：列出文件或目录的更改时间；
7.  -f：在每个文件或目录之前，显示完整的相对路径名称；
8.  -F：在执行文件，目录，Socket，符号连接，管道名称名称，各自加上"*"，"/"，"@"，"|"号；
9.  -g：列出文件或目录的所属群组名称，没有对应的名称时，则显示群组识别码；
10.  -i：不以阶梯状列出文件和目录名称；
11.  -l： 不显示符号范本样式的文件或目录名称；
12.  -l：如遇到性质为符号连接的目录，直接列出该连接所指向的原始目录；
13.  -n：不在文件和目录清单加上色彩；
14.  -N：直接列出文件和目录名称，包括控制字符；
15.  -p：列出权限标示；
16.  -P： 只显示符合范本样式的文件和目录名称；
17.  -q：用“？”号取代控制字符，列出文件和目录名称；
18.  -s：列出文件和目录大小；
19.  -t：用文件和目录的更改时间排序；
20.  -u：列出文件或目录的拥有者名称，没有对应的名称时，则显示用户识别码；
21.  -x：将范围局限在现行的文件系统中，若指定目录下的某些子目录，其存放于另一个文件系统上，则将该目录予以排除在寻找范围外。

## 设置主机名

hostnamectl set-hostname zhaohaiyu

查看:hostname

## DNS

```
DNS（Domain Name System，域名系统），万维网上作为域名和IP地址相互映射的一个分布式数据库，能够使用户更方便的访问互联网，而不用去记住能够被机器直接读取的IP数串。
通过域名，最终得到该域名对应的IP地址的过程叫做域名解析（或主机名解析）。
```

### 查看Linux的dns，唯一配置文件

```
配置文件
cat /etc/resolv.conf#dns服务器地址
nameserver 119.29.29.29
nameserver 223.5.5.5
```

### 本地强制dns解析文件/etc/hosts

```
指定本地解析：
/etc/hosts
主机IP    主机名    主机别名
127.0.0.1        www.zhaohaiyu.com
```

## yum命令

**yum命令**是在Fedora和RedHat以及SUSE中基于rpm的软件包管理器，它可以使系统管理人员交互和自动化地更细与管理RPM软件包，能够从指定的服务器自动下载RPM包并且安装，可以自动处理依赖性关系，并且一次安装所有依赖的软体包，无须繁琐地一次次下载、安装。

尽管 RPM 能够帮助用户查询软件相关的依赖关系，但问题还是要运维人员自己来解决， 而有些大型软件可能与数十个程序都有依赖关系，在这种情况下安装软件会是非常痛苦的。 Yum 软件仓库便是为了进一步降低软件安装难度和复杂度而设计的技术。Yum 软件仓库可以 根据用户的要求分析出所需软件包及其相关的依赖关系，然后自动从服务器下载软件包并安 装到系统。

Yum 软件仓库中的 RPM 软件包可以是由红帽官方发布的，也可以是第三方发布的，当 然也可以是自己编写的。

![](/images/820ddcac-811c-419c-81c1-c2214419797e.png)

选项

1.  -h：显示帮助信息；
2.  -y：对所有的提问都回答“yes”；
3.  -c：指定配置文件；
4.  -q：安静模式；
5.  -v：详细模式；
6.  -d：设置调试等级（0-10）；
7.  -e：设置错误等级（0-10）；
8.  -R：设置yum处理一个命令的最大等待时间；
9.  -C：完全从缓存中运行，而不去下载或者更新任何头文件。

实例

yum install pip

### yum源配置

yum源的目录/etc/yum.repos.d/

配置阿里云yum源

```
1.备份yum源
mkdir repo_bak
mv *.repo repo_bak/
2.下载阿里云repo文件wget http://mirrors.aliyun.com/repo/Centos-7.repo
3.清空yum缓存并且生成新的yum缓存yum clean allyum makecache4.安装软件扩展源yum install -y epel-release
yum repolist all        列出所有仓库
yum list all            列出仓库所有软件包
yum info 软件包名            查看软件包信息
yum install 软件包名        安装软件包
yum reinstall 软件包名    重新安装软件包
yum update    软件包名        升级软件包
yum remove    软件包名        移除软件包
yum clean all            清楚所有仓库缓存
yum check-update        检查可以更新的软件包
yum grouplist            查看系统中已安装的软件包
yum groupinstall 软件包组    安装软件包组
```

## Linux下安装程序的方法

*   rpm -ivh 包名.rpm　　需要手动解决依赖关系
*   yum install 包名 yum自动处理依赖关系
*   编译安装（源码安装）

### 安装Lrzsz

yum install lrzsz