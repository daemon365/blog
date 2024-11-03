---
title: "Linux常用命令"
date: "2021-08-11T00:00:00+08:00"
tags: 
- linux
showToc: true
---


## 常用指令

1. `ls`	显示文件或目录
   - `-l` 	列出文件详细信息l(list)
   - `-a`     列出当前目录下所有文件及目录，包括隐藏的a(all)

2. `mkdir`      创建目录
   - `-p`     创建目录，若无父目录，则创建p(parent)

3. `cd`            切换目录
4. `touch`       创建空文件
5. `echo`         创建带有内容的文件。
6. `cat`           查看文件内容
7. `cp`             拷贝
8. `mv`             移动或重命名
9. `rm`             删除文件
   - `-r`      递归删除，可删除子目录及文件
   - `-f`      强制删除

10. `find`         在文件系统中搜索某文件
11. `wc`         统计文本中行数、字数、字符数
12. `grep`       在文本文件中查找某个字符串
13. `rmdir`      删除空目录
14. `tree`       树形结构显示目录，需要安装tree包
15. `pwd`        显示当前目录
16. `ln`          创建链接文件
17. `more`、`less`  分页显示文本文件内容
18. `head`、`tail`   显示文件头、尾内容
19. `ctrl+alt+F1`  命令行全屏模式

 

## 系统管理命令

1. `stat`           显示指定文件的详细信息，比ls更详细
2. `who`           	显示在线登陆用户
3. `whoami`      显示当前操作用户
4. `hostname`    显示主机名
5. `uname`      显示系统信息
6. `top`         动态显示当前耗费资源最多进程信息
7. `ps`          显示瞬间进程状态 `ps -aux`
8. `du`          查看目录大小 `du -h /home`带有单位显示目录信息
9. `df`          查看磁盘大小 `df -h` 带有单位显示磁盘信息
10. `ifconfig`      查看网络情况
11. `ping`         测试网络连通
12. `netstat`      显示网络状态信息
13. `man`         命令不会用了，找男人 如：man ls
14. `clear`        清屏
15. `alias`        对命令重命名 如：`alias showmeit="ps -aux"` ，另外解除使用`unaliax showmeit`
16. `kill`         杀死进程，可以先用`ps` 或 `top`命令查看进程的id，然后再用kill命令杀死进程。

 

## 打包压缩相关命令

### gzip

- 例如：**zip -r mysql.zip mysql** 该句命令的含义是：将mysql文件夹压缩成mysql.zip
- **zip -r abcdef.zip abc def.txt** 这句命令的意思是将文件夹abc和文件def.txt压缩成一个压缩包abcdef.zip

### bzip2

- 与zip命令相反，这是解压命令，用起来很简单。 如：**unzip mysql.zip** 在当前目录下直接解压mysql.zip。

### tar

- -c        归档文件
- -x        压缩文件
- -z        gzip压缩文件
- -j        bzip2压缩文件
- -v        显示压缩或解压缩过程 v(view)
- -f        使用档名

```
例：

tar -cvf /home/abc.tar /home/abc        只打包，不压缩

tar -zcvf /home/abc.tar.gz /home/abc     打包，并用gzip压缩

tar -jcvf /home/abc.tar.bz2 /home/abc    打包，并用bzip2压缩

当然，如果想解压缩，就直接替换上面的命令 tar -cvf / tar -zcvf / tar -jcvf 中的“c” 换成“x” 就可以了。
```

## 关机/重启机器

1. `shutdown`
   - `-r`       关机重启 
   - `-h`       关机不重启
   - `now`      立刻关机

2. halt        关机

3. reboot      重启

 

## Linux管道

将一个命令的标准输出作为另一个命令的标准输入。也就是把几个命令组合起来使用，后一个命令除以前一个命令的结果。

- 例：`grep -r "close" /home/* | more`    在home目录下所有文件中查找，包括close的文件，并分页输出。

 

## Linux软件包管理



### yum常用命令

- `yum  -y install [软件包名]`　　安装
- yum erase [`软件包名`] 　　　　　卸载
- `yum clean all` 　　　　　　　   清除缓存
- `yum makecache`　　　　　　        加载缓存

### 本地yum配置

#### 本机创建yum仓库

- `mkdir -p  /root/test`
- `cd /root/test`
- `wget http://mirror.centos.org/centos-7/7/os/x86_64/Packages/dhclient-4.2.5-77.el7.centos.x86_64.rpm`

#### 生成repodata依赖文件

- `create /root/test`

#### 修改yum配置文件

1. `cd /etc/yum.repos.d`          进入yum配置目录
2. `touch local.repo`               创建配置文件
3. `vim local.repo`                   编辑配置文件

![](/images/125f240d-22d7-4a5d-bd5c-90011af5fcb6.png)

###  网络yum配置

- `cd /etc/yum.repos.d``
- ``touch cenos.repo`

![](/images/7c39be9b-b80e-4ccf-a279-f68bbe1f7d7a.png)

- `vim cenos.repo`

![](/images/5c6c19dc-b737-4ffe-aeb2-11a6d0d50ba3.png) 

**注意：baseurl路径，到repodate文件所在目录**

![](/images/9d6cd54d-11f3-4cc1-b5b1-ce9752113070.png)

###  设置本地cache保存路径

- `vim /etc/yum.conf`
  - `cachedir`       表示cache保存路径
  - `keepcache`      1-表示保存；0-表示不保

![](/images/ba6b2762-da0d-4e56-9833-9d2c1c70ce61.png)

## vim使用

vim三种模式：**命令模式、插入模式、编辑模式**。使用`ESC`或`i`或：来切换模式。

命令模式下：

1. `:q`            退出
2. `:q!`           强制退出
3. `:wq`          保存并退出
4. `:set number`   显示行号
5. `:set nonumber`  隐藏行号
6. `/apache`       在文档中查找apache 按n跳到下一个，shift+n上一个
7. `yyp`          复制光标所在行，并粘贴
8. h(左移一个字符←)、j(下一行↓)、k(上一行↑)、l(右移一个字符→)

 

## 用户及用户组管理

1. `/etc/passwd`   存储用户账号
2. `/etc/group`    存储组账号
3. `/etc/shadow`   存储用户账号的密码
4. `/etc/gshadow`  存储用户组账号的密码
5. `useradd` 用户名
6. `userdel` 用户名
7. `groupadd` 组名
8. `groupdel` 组名
9. `passwd root`   给root设置密码
10. `/etc/profile`   系统环境变量
11. `bash_profile`   用户环境变量
12. `.bashrc`        用户环境变量
13. `su user`        切换用户，加载配置文件`.bashrc`
14. `su - user`       切换用户，加载配置文件`/etc/profile` ，加载`bash_profile`

## 更改文件的用户及用户组

`sudo chown [-R] owner[:group] {File|Directory}`

```
例如：还以jdk-7u21-linux-i586.tar.gz为例。属于用户hadoop，组hadoop

要想切换此文件所属的用户及组。可以使用命令。

sudo chown root:root jdk-7u21-linux-i586.tar.gz
```

## 文件权限管理

三种基本权限

1. R       读         数值表示为4
2. W      写         数值表示为2
3. X       可执行  数值表示为1

![](/images/cf92ec89-4103-4131-866e-8e0d9e924c9c.png)


如图所示，`goblog.exe`文件的权限为`-rw-rw-rw-`

`-rw-rw-r--`一共十个字符，分成四段。

1. 第一个字符“-”表示普通文件；这个位置还可能会出现“l”链接；“d”表示目录
2. 第二三四个字符“rw-”表示当前所属用户的权限。  所以用数值表示为4+2=6
3. 第五六七个字符“rw-”表示当前所属组的权限。    所以用数值表示为4+2=6
4. 第八九十个字符“r--”表示其他用户权限。        所以用数值表示为4
5. 所以操作此文件的权限用数值表示为664 

## 更改权限

`sudo chmod [u所属用户  g所属组  o其他用户  a所有用户]  [+增加权限  -减少权限]  [r  w  x]  目录名` 

例如：有一个文件filename，权限为`-rw-r----x1` ,将权限值改为`-rwxrw-r-x`，用数值表示为765

- `sudo chmod u+x g+w o+r  filename`

上面的例子可以用数值表示

- `sudo chmod 765 filename`