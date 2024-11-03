---
title: "redis发布订阅"
date: "2022-08-20T00:00:00+08:00"
tags: 
- redis
showToc: true
---


## 什么是发布和订阅

Redis 发布订阅 (pub/sub) 是一种消息通信模式：发送者 (pub) 发送消息，订阅者 (sub) 接收消息。

Redis 客户端可以订阅任意数量的频道。

## 发布和订阅

1、客户端可以订阅频道如下图

![](/images/96a0c4ca-666f-475e-8dd0-bfbd37f4dbb6.png)

2、当给这个频道发布消息后，消息就会发送给订阅的客户端

![](/images/0c0ca636-05df-4d97-b2fe-fc3d48fba4aa.png)

## 发布订阅命令行实现

1、 打开一个客户端订阅channel1

SUBSCRIBE channel1

![](/images/8218b975-9bf5-4f05-afc9-675107cd6362.png)

2、打开另一个客户端，给channel1发布消息hello

publish channel1 hello

![](/images/3948289f-dcd0-4f46-8b0d-b286f7cee2dd.png)

返回的1是订阅者数量

3、打开第一个客户端可以看到发送的消息

![](/images/5196ef36-5138-45a4-b6d4-b299b4dc5933.png)

注：发布的消息没有持久化，如果在订阅的客户端收不到hello，只能收到订阅后发布的消息