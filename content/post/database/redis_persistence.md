---
title: redis持久化
date: '2020-03-21T00:00:00+08:00'
tags:
- redis
showToc: true
categories:
- database
---



Redis是一种内存型数据库，一旦服务器进程退出，数据库的数据就会丢失，为了解决这个问题，Redis提供了两种持久化的方案，将内存中的数据保存到磁盘中，避免数据的丢失。

## RDB持久化

redis提供了RDB持久化的功能，这个功能可以将redis在内存中的的状态保存到硬盘中，它可以**手动执行。**

也可以再redis.conf中配置，**定期执行**。

RDB持久化产生的RDB文件是一个**经过压缩**的**二进制文件**，这个文件被保存在硬盘中，redis可以通过这个文件还原数据库当时的状态。

*   内存数据保存到磁盘

*   在指定的时间间隔内生成数据集的时间点快照（point-in-time snapshot）

*   优点：速度快，适合做备份，主从复制就是基于RDB持久化功能实现

*   rdb通过再redis中使用save命令触发 rdb


1. rdb配置参数： `dir /data/6379/ dbfilename dbmp.rdb` 
2. 每过900秒 有1个操作就进行持久化 save 900秒 1个修改类的操作 save 300秒 10个操作 save 60秒 10000个操作  `save 900 1 save 300 10 save 60 10000`


### redis持久化之RDB实践

1.启动redis服务端,准备配置文件

```bash
daemonize yes port 6379 logfile /data/6379/redis.log dir /data/6379    #定义持久化文件存储位置 
dbfilename dbmp.rdb        #rdb持久化文件 
bind 10.0.0.10 127.0.0.1   #redis绑定地址 
requirepass redhat         #redis登录密码 
save 900 1                 #rdb机制 每900秒 有1个修改记录 
save 300 10                #每300秒       10个修改记录 
save 60 10000              #每60秒内     10000修改记录
```

2.启动redis服务端

3.登录redis设置一个key

```bash
redis-cli -a redhat
```

4.此时检查目录，/data/6379底下没有dbmp.rdb文件

5.通过save触发持久化，将数据写入RDB文件

```bash
127.0.0.1:6379> set age 18 OK 127.0.0.1:6379> save OK
```

## AOF持久化

AOF（append-only log file）

*   记录服务器执行的所有变更操作命令（例如set del等），并在服务器启动时，通过重新执行这些命令来还原数据集

*   AOF 文件中的命令全部以redis协议的格式保存，新命令追加到文件末尾。

*   优点：最大程序保证数据不丢

*   缺点：日志记录非常大

```bash
redis-client   写入数据  >  redis-server   同步命令   >  AOF文件
```

配置参数

```
AOF持久化配置，两条参数  
appendonly yes appendfsync always   总是修改类的操作  
everysec   每秒做一次持久化  no     依赖于系统自带的缓存大小机制
```

### redis持久化之AOF实践

1.准备aof配置文件 redis.conf

```
daemonize yes
port 6379 
logfile /data/6379/redis.log 
dir /data/6379 
dbfilename dbmp.rdb 
requirepass redhat 
save 900 1 
save 300 10 
save 60 10000 
appendonly yes 
appendfsync everysec
```

2.启动redis服务

```
redis-server /etc/redis.conf
```

3.检查redis数据目录/data/6379/是否产生了aof文件

```
[root@web02 6379]## ls appendonly.aof dbmp.rdb redis.log
```

4.登录redis-cli，写入数据，实时检查aof文件信息

```
[root@web02 6379]## tail -f appendonly.aof
```

5.设置新key，检查aof信息，然后关闭redis，检查数据是否持久化

```
redis-cli -a redhat shutdown  redis-server /etc/redis.conf  redis-cli -a redhat
```

## redis 持久化方式有哪些？有什么区别

*   rdb：基于快照的持久化，速度更快，一般用作备份，主从复制也是依赖于rdb持久化功能

*   aof：以追加的方式记录redis操作日志的文件。可以最大程度的保证redis数据安全，类似于mysql的binlog

## redis不重启，切换RDB备份到AOF备份

### 确保redis版本在2.2以上

```
[root@pyyuc /data 22:23:30]#redis-server -v
Redis server v=4.0.10 sha=00000000:0 malloc=jemalloc-4.0.3 bits=64 build=64cb6afcf41664c
```

本文在redis4.0中，通过config set命令，达到不重启redis服务，从RDB持久化切换为AOF

### 实验环境准备

redis.conf服务端配置文件

```ini
daemonize yes
port 6379
logfile /data/6379/redis.log
dir /data/6379
dbfilename  dbmp.rdb
save 900 1                    #rdb机制 每900秒 有1个修改记录
save 300 10                    #每300秒        10个修改记录
save 60  10000                #每60秒内        10000修改记录
```

启动redis服务端

```ini
redis-server redis.conf
```

登录redis-cli插入数据，手动持久化

```bash
127.0.0.1:6379> set name chaoge
OK
127.0.0.1:6379> set age 18
OK
127.0.0.1:6379> set addr shahe
OK
127.0.0.1:6379> save
OK
```

检查RDB文件

```bash
[root@pyyuc /data 22:34:16]#ls 6379/
dbmp.rdb  redis.log
```

### 备份这个rdb文件，保证数据安全

```bash
[root@pyyuc /data/6379 22:35:38]#cp dbmp.rdb /opt/
```

### 执行命令，开启AOF持久化

```bash
127.0.0.1:6379> CONFIG set appendonly yes   #开启AOF功能
OK
127.0.0.1:6379> CONFIG SET save ""  #关闭RDB功能
OK
```

### 确保数据库的key数量正确

```bash
127.0.0.1:6379> keys *
1) "addr"
2) "age"
3) "name"
```

### 确保插入新的key，AOF文件会记录

```bash
127.0.0.1:6379> set title golang
OK
```

**此时RDB已经正确切换AOF，注意还得修改redis.conf添加AOF设置，不然重启后，通过config set的配置将丢失**