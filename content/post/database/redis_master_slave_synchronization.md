---
title: redis主从同步
date: '2023-08-20T00:00:00+08:00'
tags:
- redis
showToc: true
categories:
- database
---



## redis主从同步

![](/images/a9aeda50-50ab-4fe3-b94c-572cadf14613.png)

原理：

1.  从服务器向主服务器发送 SYNC 命令。

2.  接到 SYNC 命令的主服务器会调用BGSAVE 命令，创建一个 RDB 文件，并使用缓冲区记录接下来执行的所有写命令。

3.  当主服务器执行完 BGSAVE 命令时，它会向从服务器发送 RDB 文件，而从服务器则会接收并载入这个文件。

4.  主服务器将缓冲区储存的所有写命令发送给从服务器执行。

------------- 1、在开启主从复制的时候，使用的是RDB方式的，同步主从数据的 2、同步开始之后，通过主库命令传播的方式，主动的复制方式实现 3、2.8以后实现PSYNC的机制，实现断线重连

## 环境准备

6380.conf

```
1、环境： 准备两个或两个以上redis实例  mkdir /data/638{0..2} #创建6380 6381 6382文件夹  配置文件示例： vim   /data/6380/redis.conf port 6380 daemonize yes pidfile /data/6380/redis.pid loglevel notice logfile "/data/6380/redis.log" dbfilename dump.rdb dir /data/6380 protected-mode no
```

6381.conf

```
vim   /data/6381/redis.conf port 6381 daemonize yes pidfile /data/6381/redis.pid loglevel notice logfile "/data/6381/redis.log" dbfilename dump.rdb dir /data/6381 protected-mode no
```

6382.conf

```
port 6382 daemonize yes pidfile /data/6382/redis.pid loglevel notice logfile "/data/6382/redis.log" dbfilename dump.rdb dir /data/6382 protected-mode no
```

启动三个redis实例

```
redis-server /data/6380/redis.conf redis-server /data/6381/redis.conf redis-server /data/6382/redis.conf
```

主从规划

```
主节点：6380 从节点：6381、6382
```

## 配置主从同步

6381/6382命令行

redis-cli -p 6381 SLAVEOF 127.0.0.1 6380 #指明主的地址

redis-cli -p 6382 SLAVEOF 127.0.0.1 6380 #指明主的地址

检查主从状态

从库：

```
127.0.0.1:6382> info replication 127.0.0.1:6381> info replication
```

主库：

```
127.0.0.1:6380> info replication
```

## 测试写入数据，主库写入数据，检查从库数据

```
主 127.0.0.1:6380> set name chaoge   从 127.0.0.1:6381>get name 
```

## 手动进行主从复制故障切换

```
#关闭主库6380redis-cli -p 6380 shutdown
```

检查从库主从信息，此时master_link_status:down

```
redis-cli -p 6381 info replication redis-cli -p 6382 info replication
```

**既然主库挂了，我想要在6381 6382之间选一个新的主库**

**1.关闭6381的从库身份**

```
redis-cli -p 6381 info replication slaveof no one
```

**2.将6382设为6381的从库**

```
6382连接到6381： [root@db03 ~]## redis-cli -p 6382 127.0.0.1:6382> SLAVEOF no one 127.0.0.1:6382> SLAVEOF 127.0.0.1 6381
```

3.检查6382，6381的主从信息