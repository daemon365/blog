---
title: "网络编程"
date: "2019-12-21T00:00:00+08:00"
tags: 
- network
showToc: true
---


## 网络层次划分

为了使不同计算机厂家生产的计算机能够相互通信，以便在更大的范围内建立计算机网络，国际标准化组织（ISO）在1978年提出了"开放系统互联参考模型"，即著名的OSI/RM模型（Open System Interconnection/Reference Model）。它将计算机网络体系结构的通信协议划分为七层，自下而上依次为：物理层（Physics Layer）、数据链路层（Data Link Layer）、网络层（Network Layer）、传输层（Transport Layer）、会话层（Session Layer）、表示层（Presentation Layer）、应用层（Application Layer）。其中第四层完成数据传送服务，上面三层面向用户。

## osi七层协议

- 互联网协议按照功能不同分为osi七层或tcp/ip五层

![](/images/e1eba343-af03-4c75-a1d5-97b18230cad2.png)

### 物理层

物理层由来：上面提到，孤立的计算机之间要通信，就必须接入internet

![](/images/118e6820-bb27-4651-b8a1-d86a26c32e27.png)

**物理层功能：主要是基于电器特性发送高低电压(电信号)，高电压对应数字1，低电压对应数字0**

光纤： 双绞线：

![](/images/0a344786-7492-4815-8043-c3a60013b4cc.png)![](/images/9e604851-7c87-4b33-8c3a-04d181111f69.png)

### 数据链路层

数据链路层由来：单纯的电信号0和1没有任何意义，必须规定电信号多少位一组，每组什么意思

数据链路层的功能：定义了电信号的分组方式

***以太网协议：***

- 早期的时候各个公司都有自己的分组方式，后来形成了统一的标准，即以太网协议ethernet

ethernet规定

- 一组电信号构成一个数据豹，叫做‘帧’
- 每一数据帧分成：报头head和数据data两部分

1. head包含：(固定18个字节)
   - 发送者／源地址，6个字节
   - 接收者／目标地址，6个字节
   - 数据类型，6个字节
2. data包含：(最短46字节，最长1500字节)
   - 数据包的具体内容
   - head长度＋data长度＝最短64字节，最长1518字节，超过最大限制就分片发送

***mac地址：***

每块网卡出厂时都被烧制上一个世界唯一的mac地址，长度为48位2进制，通常由12位16进制数表示（前六位是厂商编号，后六位是流水线号）

![](/images/2af10484-1932-482f-93fb-863f857702fd.png)

**广播：**

有了mac地址，同一网络内的两台主机就可以通信了（一台主机通过arp协议获取另外一台主机的mac地址）

ethernet采用最原始的方式，广播的方式进行通信，即计算机通信***

![](/images/ee7fe03b-2d47-4115-8c45-37b613b317f2.png)

### 网络层

网络层由来：有了ethernet、mac地址、广播的发送方式，世界上的计算机就可以彼此通信了，问题是世界范围的互联网是由

一个个彼此隔离的小的局域网组成的，那么如果所有的通信都采用以太网的广播方式，那么一台机器发送的包全世界都会收到，

这就不仅仅是效率低的问题了，这会是一种灾难

![](/images/920d1763-9497-4e14-a384-e29d0acc91f6.png)

上图结论：必须找出一种方法来区分哪些计算机属于同一广播域，哪些不是，如果是就采用广播的方式发送，如果不是，

就采用路由的方式（向不同广播域／子网分发数据包），mac地址是无法区分的，它只跟厂商有关

**网络层功能：引入一套新的地址用来区分不同的广播域／子网，这套地址即网络地址**

***IP协议：***

- 规定网络地址的协议叫ip协议，它定义的地址称之为ip地址，广泛采用的v4版本即ipv4，它规定网络地址由32位2进制表示
- 范围0.0.0.0-255.255.255.255
- 一个ip地址通常写成四段十进制数，例：172.16.10.1

**ip地址分成两部分**

- 网络部分：标识子网
- 主机部分：标识主机

注意：单纯的ip地址段只是标识了ip地址的种类，从网络部分或主机部分都无法辨识一个ip所处的子网

例：172.16.10.1与172.16.10.2并不能确定二者处于同一子网

***子网掩码***

所谓”子网掩码”，就是表示子网络特征的一个参数。它在形式上等同于IP地址，也是一个32位二进制数字，它的网络部分全部为1，主机部分全部为0。比如，IP地址172.16.10.1，如果已知网络部分是前24位，主机部分是后8位，那么子网络掩码就是11111111.11111111.11111111.00000000，写成十进制就是255.255.255.0。

知道”子网掩码”，我们就能判断，任意两个IP地址是否处在同一个子网络。方法是将两个IP地址与子网掩码分别进行AND运算（两个数位都为1，运算结果为1，否则为0），然后比较结果是否相同，如果是的话，就表明它们在同一个子网络中，否则就不是。

比如，已知IP地址172.16.10.1和172.16.10.2的子网掩码都是255.255.255.0，请问它们是否在同一个子网络？两者与子网掩码分别进行AND运算，

172.16.10.1：10101100.00010000.00001010.000000001

255255.255.255.0:11111111.11111111.11111111.00000000

AND运算得网络地址结果：10101100.00010000.00001010.000000001->172.16.10.0

172.16.10.2：10101100.00010000.00001010.000000010

255255.255.255.0:11111111.11111111.11111111.00000000

AND运算得网络地址结果：10101100.00010000.00001010.000000001->172.16.10.0

结果都是172.16.10.0，因此它们在同一个子网络。

总结一下，IP协议的作用主要有两个，一个是为每一台计算机分配IP地址，另一个是确定哪些地址在同一个子网络。

***ip数据包***

ip数据包也分为head和data部分，无须为ip包定义单独的栏位，直接放入以太网包的data部分

head：长度为20到60字节

data：最长为65,515字节。

而以太网数据包的”数据”部分，最长只有1500字节。因此，如果IP数据包超过了1500字节，它就需要分割成几个以太网数据包，分开发送了。

***ARP协议***

arp协议由来：计算机通信***，即广播的方式，所有上层的包到最后都要封装上以太网头，然后通过以太网协议发送，在谈及以太网协议时候，我门了解到

通信是基于mac的广播方式实现，计算机在发包时，获取自身的mac是容易的，如何获取目标主机的mac，就需要通过arp协议

arp协议功能：广播的方式发送数据包，获取目标主机的mac地址

协议工作方式：每台主机ip都是已知的

例如：主机172.16.10.10/24访问172.16.10.11/24

一：首先通过ip地址和子网掩码区分出自己所处的子网

| 场景     | 数据包地址              |
| -------- | ----------------------- |
| 同一子网 | 目标主机mac，目标主机ip |
| 不同子网 | 网关mac，目标主机ip     |

二：分析172.16.10.10/24与172.16.10.11/24处于同一网络(如果不是同一网络，那么下表中目标ip为172.16.10.1,通过arp获取的是网关的mac)

| 源mac      | 目标mac   | 源ip              | 目标ip          | 数据部分        |      |
| ---------- | --------- | ----------------- | --------------- | --------------- | ---- |
| 发送端主机 | 发送端mac | FF:FF:FF:FF:FF:FF | 172.16.10.10/24 | 172.16.10.11/24 | 数据 |

三：这个包会以广播的方式在发送端所处的自网内传输，所有主机接收后拆开包，发现目标ip为自己的，就响应，返回自己的mac

### 传输层

传输层的由来：网络层的ip帮我们区分子网，以太网层的mac帮我们找到主机，然后大家使用的都是应用程序，你的电脑上可能同时开启qq，暴风影音，等多个应用程序，

那么我们通过ip和mac找到了一台特定的主机，如何标识这台主机上的应用程序，答案就是端口，端口即应用程序与网卡关联的编号。

传输层功能：建立端口到端口的通信

补充：端口范围0-65535，0-1023为系统占用端口

tcp协议：

可靠传输，TCP数据包没有长度限制，理论上可以无限长，但是为了保证网络的效率，通常TCP数据包的长度不会超过IP数据包的长度，以确保单个TCP数据包不必再分割。

| 以太网头 | ip 头 | tcp头 | 数据      |
| -------- | ----- | ----- | --------- |
|          |       |       | udp协议： |

不可靠传输，”报头”部分一共只有8个字节，总长度不超过65,535字节，正好放进一个IP数据包。

| 以太网头 | ip头 | udp头 | 数据 |
| -------- | ---- | ----- | ---- |
|          |      |       |      |

tcp报文

![](/images/757a5ec0-36e3-4fc2-95f8-bd2afc1a8f20.png)

**tcp三次握手和四次挥手**

![](/images/15eb11ae-7c93-445b-9809-4d4dd3840b09.png)

![](/images/73d687e1-618f-4a40-bce2-29277c4d9207.png)

### 应用层

应用层由来：用户使用的都是应用程序，均工作于应用层，互联网是开发的，大家都可以开发自己的应用程序，数据多种多样，必须规定好数据的组织形式

应用层功能：规定应用程序的数据格式。

例：TCP协议可以为各种各样的程序传递数据，比如Email、WWW、FTP等等。那么，必须有不同协议规定电子邮件、网页、FTP数据的格式，这些应用程序协议就构成了”应用层”。

![](/images/488cc59e-6f66-41a0-ace6-873d1e44b0fd.png)

### 网络通信实现

想实现网络通信，每台主机需具备四要素

- 本机的IP地址
- 子网掩码
- 网关的IP地址
- DNS的IP地址

获取这四要素分两种方式

1.静态获取

即手动配置

2.动态获取

通过dhcp获取

| 以太网头 | ip头 | udp头 | dhcp数据包 |
| -------- | ---- | ----- | ---------- |
|          |      |       |            |

（1）最前面的”以太网标头”，设置发出方（本机）的MAC地址和接收方（DHCP服务器）的MAC地址。前者就是本机网卡的MAC地址，后者这时不知道，就填入一个广播地址：FF-FF-FF-FF-FF-FF。

（2）后面的”IP标头”，设置发出方的IP地址和接收方的IP地址。这时，对于这两者，本机都不知道。于是，发出方的IP地址就设为0.0.0.0，接收方的IP地址设为255.255.255.255。

（3）最后的”UDP标头”，设置发出方的端口和接收方的端口。这一部分是DHCP协议规定好的，发出方是68端口，接收方是67端口。

这个据包构造完成后，就可以发出了。以太网是广播发送，同一个子网络的每台计算机都收到了这个包。因为接收方的MAC地址是FF-FF-FF-FF-FF-FF，看不出是发给谁的，所以每台收到这个包的计算机，还必须分析这个包的IP地址，才能确定是不是发给自己的。当看到发出方IP地址是0.0.0.0，接收方是255.255.255.255，于是DHCP服务器知道”这个包是发给我的”，而其他计算机就可以丢弃这个包。

接下来，DHCP服务器读出这个包的数据内容，分配好IP地址，发送回去一个”DHCP响应”数据包。这个响应包的结构也是类似的，以太网标头的MAC地址是双方的网卡地址，IP标头的IP地址是DHCP服务器的IP地址（发出方）和255.255.255.255（接收方），UDP标头的端口是67（发出方）和68（接收方），分配给请求端的IP地址和本网络的具体参数则包含在Data部分。新加入的计算机收到这个响应包，于是就知道了自己的IP地址、子网掩码、网关地址、DNS服务器等等参数

### 网络通信流程

#### 本机获取

- 本机的IP地址：192.168.1.100
- 子网掩码：255.255.255.0
- 网关的IP地址：192.168.1.1
- DNS的IP地址：8.8.8.8

#### 打开浏览器，访问

- 想要访问Google，在地址栏输入了网址：www.google.com。

### dns协议(基于udp协议)

![](/images/a53369a0-b98e-45dd-a40d-cb9842a20c28.png)

13台根dns：

A.root-servers.net198.41.0.4美国 B.root-servers.net192.228.79.201美国（另支持[IPv6](https://www.baidu.com/s?wd=IPv6&tn=44039180_cpr&fenlei=mv6quAkxTZn0IZRqIHckPjm4nH00T1d9ryR4nhnsn1b3ujfYPAfk0ZwV5Hcvrjm3rH6sPfKWUMw85HfYnjn4nH6sgvPsT6KdThsqpZwYTjCEQLGCpyw9Uz4Bmy-bIi4WUvYETgN-TLwGUv3ErH0YPW6snH0)） C.root-servers.net192.33.4.12法国 D.root-servers.net128.8.10.90美国 E.root-servers.net192.203.230.10美国 F.root-servers.net192.5.5.241美国（另支持[IPv6](https://www.baidu.com/s?wd=IPv6&tn=44039180_cpr&fenlei=mv6quAkxTZn0IZRqIHckPjm4nH00T1d9ryR4nhnsn1b3ujfYPAfk0ZwV5Hcvrjm3rH6sPfKWUMw85HfYnjn4nH6sgvPsT6KdThsqpZwYTjCEQLGCpyw9Uz4Bmy-bIi4WUvYETgN-TLwGUv3ErH0YPW6snH0)） G.root-servers.net192.112.36.4美国 H.root-servers.net128.63.2.53美国（另支持[IPv6](https://www.baidu.com/s?wd=IPv6&tn=44039180_cpr&fenlei=mv6quAkxTZn0IZRqIHckPjm4nH00T1d9ryR4nhnsn1b3ujfYPAfk0ZwV5Hcvrjm3rH6sPfKWUMw85HfYnjn4nH6sgvPsT6KdThsqpZwYTjCEQLGCpyw9Uz4Bmy-bIi4WUvYETgN-TLwGUv3ErH0YPW6snH0)） I.root-servers.net192.36.148.17瑞典 J.root-servers.net192.58.128.30美国 K.root-servers.net193.0.14.129英国（另支持IPv6） L.root-servers.net198.32.64.12美国 M.root-servers.net202.12.27.33日本（另支持IPv6）

域名定义：http://jingyan.baidu.com/article/1974b289a649daf4b1f774cb.html

顶级域名：以.com,.net,.org,.cn等等属于国际顶级域名，根据目前的国际互联网域名体系，国际顶级域名分为两类：类别顶级域名(gTLD)和地理顶级域名(ccTLD)两种。类别顶级域名是　　　　　　　　 以"COM"、"NET"、"ORG"、"BIZ"、"INFO"等结尾的域名，均由国外公司负责管理。地理顶级域名是以国家或地区代码为结尾的域名，如"CN"代表中国，"UK"代表英国。地理顶级域名一般由各个国家或地区负责管理。

二级域名：二级域名是以顶级域名为基础的地理域名，比喻中国的二级域有，.com.cn,.net.cn,.org.cn,.gd.cn等.子域名是其父域名的子域名，比喻父域名是abc.com,子域名就是www.abc.com或者*.abc.com. 一般来说，二级域名是域名的一条记录，比如alidiedie.com是一个域名，www.alidiedie.com是其中比较常用的记录，一般默认是用这个，但是类似*.alidiedie.com的域名全部称作是alidiedie.com的二级

#### HTTP部分的内容

类似于这样的：

> GET / HTTP/1.1 Host: [www.google.com](http://www.google.com/) Connection: keep-alive User-Agent: Mozilla/5.0 (Windows NT 6.1) …… Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8 Accept-Encoding: gzip,deflate,sdch Accept-Language: zh-CN,zh;q=0.8 Accept-Charset: GBK,utf-8;q=0.7,*;q=0.3 Cookie: … …

我们假定这个部分的长度为4960字节，它会被嵌在TCP数据包之中。

#### TCP协议

TCP数据包需要设置端口，接收方（Google）的HTTP端口默认是80，发送方（本机）的端口是一个随机生成的1024-65535之间的整数，假定为51775。

TCP数据包的标头长度为20字节，加上嵌入HTTP的数据包，总长度变为4980字节。

#### IP协议

然后，TCP数据包再嵌入IP数据包。IP数据包需要设置双方的IP地址，这是已知的，发送方是192.168.1.100（本机），接收方是172.194.72.105（Google）。

IP数据包的标头长度为20字节，加上嵌入的TCP数据包，总长度变为5000字节。

#### 以太网协议

最后，IP数据包嵌入以太网数据包。以太网数据包需要设置双方的MAC地址，发送方为本机的网卡MAC地址，接收方为网关192.168.1.1的MAC地址（通过ARP协议得到）。

以太网数据包的数据部分，最大长度为1500字节，而现在的IP数据包长度为5000字节。因此，IP数据包必须分割成四个包。因为每个包都有自己的IP标头（20字节），所以四个包的IP数据包的长度分别为1500、1500、1500、560。

#### 服务器端响应

经过多个网关的转发，Google的服务器172.194.72.105，收到了这四个以太网数据包。

根据IP标头的序号，Google将四个包拼起来，取出完整的TCP数据包，然后读出里面的”HTTP请求”，接着做出”HTTP响应”，再用TCP协议发回来。

本机收到HTTP响应以后，就可以将网页显示出来，完成一次网络通信。

#### socket

五层通讯流程：

![](/images/d143b8c3-0b62-4cbf-9e89-9f8c644613fc.png)

但实际上从传输层开始以及以下，都是操作系统帮咱们完成的，下面的各种包头封装的过程，用咱们去一个一个做么？**NO！**

![](/images/23a994b7-a894-48e9-953c-c2b83a08da6e.png)

Socket又称为套接字，它是应用层与TCP/IP协议族通信的中间软件抽象层，它是一组接口。在设计模式中，Socket其实就是一个门面模式，它把复杂的TCP/IP协议族隐藏在Socket接口后面，对用户来说，一组简单的接口就是全部，让Socket去组织数据，以符合指定的协议。当我们使用不同的协议进行通信时就得使用不同的接口，还得处理不同协议的各种细节，这就增加了开发的难度，软件也不易于扩展(就像我们开发一套公司管理系统一样，报账、会议预定、请假等功能不需要单独写系统，而是一个系统上多个功能接口，不需要知道每个功能如何去实现的)。于是UNIX BSD就发明了socket这种东西，socket屏蔽了各个协议的通信细节，使得程序员无需关注协议本身，直接使用socket提供的接口来进行互联的不同主机间的进程的通信。这就好比操作系统给我们提供了使用底层硬件功能的系统调用，通过系统调用我们可以方便的使用磁盘（文件操作），使用内存，而无需自己去进行磁盘读写，内存管理。socket其实也是一样的东西，就是提供了tcp/ip协议的抽象，对外提供了一套接口，同过这个接口就可以统一、方便的使用tcp/ip协议的功能了。

其实站在你的角度上看，socket就是一个模块。我们通过调用模块中已经实现的方法建立两个进程之间的连接和通信。也有人将socket说成ip+port，因为ip是用来标识互联网中的一台主机的位置，而port是用来标识这台机器上的一个应用程序。 所以我们只要确立了ip和port就能找到一个应用程序，并且使用socket模块来与之通信。

#### 套接字发展史及分类

套接字起源于 20 世纪 70 年代加利福尼亚大学伯克利分校版本的 Unix,即人们所说的 BSD Unix。 因此,有时人们也把套接字称为“伯克利套接字”或“BSD 套接字”。一开始,套接字被设计用在同 一台主机上多个应用程序之间的通讯。这也被称进程间通讯,或 IPC。套接字有两种（或者称为有两个种族）,分别是基于文件型的和基于网络型的。

***基于文件类型的套接字家族***

套接字家族的名字：AF_UNIX

unix一切皆文件，基于文件的套接字调用的就是底层的文件系统来取数据，两个套接字进程运行在同一机器，可以通过访问同一个文件系统间接完成通信

***基于网络类型的套接字家族***

套接字家族的名字：AF_INET

(还有AF_INET6被用于ipv6，还有一些其他的地址家族，不过，他们要么是只用于某个平台，要么就是已经被废弃，或者是很少被使用，或者是根本没有实现，所有地址家族中，AF_INET是使用最广泛的一个，python支持很多种地址家族，但是由于我们只关心网络编程，所以大部分时候我么只使用AF_INET)

#### 套接字的工作流程（基于TCP和 UDP两个协议）

##### TCP和UDP对比

TCP（Transmission Control Protocol）可靠的、面向连接的协议（eg:打电话）、传输效率低全双工通信（发送缓存&接收缓存）、面向字节流。使用TCP的应用：Web浏览器；文件传输程序。

UDP（User Datagram Protocol）不可靠的、无连接的服务，传输效率高（发送前时延小），一对一、一对多、多对一、多对多、面向报文(数据包)，尽最大努力服务，无拥塞控制。使用UDP的应用：域名系统 (DNS)；视频流；IP语音(VoIP)。

##### TCP协议下的socket

个生活中的场景。你要打电话给一个朋友，先拨号，朋友听到电话铃声后提起电话，这时你和你的朋友就建立起了连接，就可以讲话了。等交流结束，挂断电话结束此次交谈。 生活中的场景就解释了这工作原理。

![](/images/b92b90bc-d7eb-4302-a3ad-6447442453d2.png)

先从服务器端说起。服务器端先初始化Socket，然后与端口绑定(bind)，对端口进行监听(listen)，调用accept阻塞，等待客户端连接。在这时如果有个客户端初始化一个Socket，然后连接服务器(connect)，如果连接成功，这时客户端与服务器端的连接就建立了。客户端发送数据请求，服务器端接收请求并处理请求，然后把回应数据发送给客户端，客户端读取数据，最后关闭连接，一次交互结束 细说socket()模块函数用法

```python
import socket
socket.socket(socket_family,socket_type,protocal=0)
 socket_family 可以是 AF_UNIX 或 AF_INET。socket_type 可以是 SOCK_STREAM 或 SOCK_DGRAM。protocol 一般不填,默认值为 0。

 获取tcp/ip套接字
tcpSock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

获取udp/ip套接字
udpSock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

由于 socket 模块中有太多的属性。我们在这里破例使用了'from module import *'语句。使用 'from socket import *',我们就把 socket 模块里的所有属性都带到我们的命名空间里了,这样能 大幅减短我们的代码。
例如tcpSock = socket(AF_INET, SOCK_STREAM)
服务端套接字函数
s.bind()    绑定(主机,端口号)到套接字
s.listen()  开始TCP监听
s.accept()  被动接受TCP客户的连接,(阻塞式)等待连接的到来

客户端套接字函数
s.connect()     主动初始化TCP服务器连接
s.connect_ex()  connect()函数的扩展版本,出错时返回出错码,而不是抛出异常

公共用途的套接字函数
s.recv()            接收TCP数据
s.send()            发送TCP数据(send在待发送数据量大于己端缓存区剩余空间时,数据丢失,不会发完)
s.sendall()         发送完整的TCP数据(本质就是循环调用send,sendall在待发送数据量大于己端缓存区剩余空间时,数据不丢失,循环调用send直到发完)
s.recvfrom()        接收UDP数据
s.sendto()          发送UDP数据
s.getpeername()     连接到当前套接字的远端的地址
s.getsockname()     当前套接字的地址
s.getsockopt()      返回指定套接字的参数
s.setsockopt()      设置指定套接字的参数
s.close()           关闭套接字

面向锁的套接字方法
s.setblocking()     设置套接字的阻塞与非阻塞模式
s.settimeout()      设置阻塞套接字操作的超时时间
s.gettimeout()      得到阻塞套接字操作的超时时间

面向文件的套接字的函数
s.fileno()          套接字的文件描述符
s.makefile()        创建一个与该套接字相关的文件
```

第一版，单个客户端与服务端通信（low版）

```python
import socket

phone = socket.socket(socket.AF_INET,socket.SOCK_STREAM)  # 买电话

phone.connect(('127.0.0.1',8080))  # 与客户端建立连接， 拨号

phone.send('hello'.encode('utf-8'))

from_server_data = phone.recv(1024)

print(from_server_data)

phone.close()  # 挂电话
# 网络通信与打电话（诺基亚）是一样的。

import socket

phone = socket.socket(socket.AF_INET,socket.SOCK_STREAM)  # 买电话

phone.bind(('127.0.0.1',8080))  # 0 ~ 65535  1024之前系统分配好的端口 绑定电话卡

phone.listen(5)  # 同一时刻有5个请求，但是可以有N多个链接。 开机。


conn, client_addr = phone.accept()  # 接电话
print(conn, client_addr, sep='\n')

from_client_data = conn.recv(1024)  # 一次接收的最大限制  bytes
print(from_client_data.decode('utf-8'))

conn.send(from_client_data.upper())

conn.close()  # 挂电话

phone.close() # 关机

服务端
```

第二版，通信循环

```python
import socket

phone = socket.socket(socket.AF_INET,socket.SOCK_STREAM)

phone.bind(('127.0.0.1',8080))

phone.listen(5)


conn, client_addr = phone.accept()
print(conn, client_addr, sep='\n')

while 1:  # 循环收发消息
    try:
        from_client_data = conn.recv(1024)
        print(from_client_data.decode('utf-8'))
    
        conn.send(from_client_data + b'SB')
    
    except ConnectionResetError:
        break

conn.close()
phone.close()
import socket

phone = socket.socket(socket.AF_INET,socket.SOCK_STREAM)  # 买电话

phone.connect(('127.0.0.1',8080))  # 与客户端建立连接， 拨号


while 1:  # 循环收发消息
    client_data = input('>>>')
    phone.send(client_data.encode('utf-8'))
    
    from_server_data = phone.recv(1024)
    
    print(from_server_data.decode('utf-8'))

phone.close()  # 挂电话
```

第三版， 通信，连接循环

```python
import socket

phone = socket.socket(socket.AF_INET,socket.SOCK_STREAM)

phone.bind(('127.0.0.1',8080))

phone.listen(5)

while 1 : # 循环连接客户端
    conn, client_addr = phone.accept()
    print(client_addr)
    
    while 1:
        try:
            from_client_data = conn.recv(1024)
            print(from_client_data.decode('utf-8'))
        
            conn.send(from_client_data + b'SB')
        
        except ConnectionResetError:
            break

conn.close()
phone.close()
import socket

phone = socket.socket(socket.AF_INET,socket.SOCK_STREAM)  # 买电话

phone.connect(('127.0.0.1',8080))  # 与客户端建立连接， 拨号


while 1:
    client_data = input('>>>')
    phone.send(client_data.encode('utf-8'))
    
    from_server_data = phone.recv(1024)
    
    print(from_server_data.decode('utf-8'))

phone.close()  # 挂电话
```

***详解recv的工作原理***

```python
'''
源码解释：
Receive up to buffersize bytes from the socket.
接收来自socket缓冲区的字节数据，
For the optional flags argument, see the Unix manual.
对于这些设置的参数，可以查看Unix手册。
When no data is available, block untilat least one byte is available or until the remote end is closed.
当缓冲区没有数据可取时，recv会一直处于阻塞状态，直到缓冲区至少有一个字节数据可取，或者远程端关闭。
When the remote end is closed and all data is read, return the empty string.
关闭远程端并读取所有数据后，返回空字符串。
'''
----------服务端------------：
# 1，验证服务端缓冲区数据没有取完，又执行了recv执行，recv会继续取值。

import socket

phone =socket.socket(socket.AF_INET,socket.SOCK_STREAM)

phone.bind(('127.0.0.1',8080))

phone.listen(5)

conn, client_addr = phone.accept()
from_client_data1 = conn.recv(2)
print(from_client_data1)
from_client_data2 = conn.recv(2)
print(from_client_data2)
from_client_data3 = conn.recv(1)
print(from_client_data3)
conn.close()
phone.close()

# 2，验证服务端缓冲区取完了，又执行了recv执行，此时客户端20秒内不关闭的前提下，recv处于阻塞状态。

import socket

phone =socket.socket(socket.AF_INET,socket.SOCK_STREAM)

phone.bind(('127.0.0.1',8080))

phone.listen(5)

conn, client_addr = phone.accept()
from_client_data = conn.recv(1024)
print(from_client_data)
print(111)
conn.recv(1024) # 此时程序阻塞20秒左右，因为缓冲区的数据取完了，并且20秒内，客户端没有关闭。
print(222)

conn.close()
phone.close()


# 3 验证服务端缓冲区取完了，又执行了recv执行，此时客户端处于关闭状态，则recv会取到空字符串。

import socket

phone =socket.socket(socket.AF_INET,socket.SOCK_STREAM)

phone.bind(('127.0.0.1',8080))

phone.listen(5)

conn, client_addr = phone.accept()
from_client_data1 = conn.recv(1024)
print(from_client_data1)
from_client_data2 = conn.recv(1024)
print(from_client_data2)
from_client_data3 = conn.recv(1024)
print(from_client_data3)
conn.close()
phone.close()
------------客户端------------
# 1，验证服务端缓冲区数据没有取完，又执行了recv执行，recv会继续取值。
import socket
import time
phone = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
phone.connect(('127.0.0.1',8080))
phone.send('hello'.encode('utf-8'))
time.sleep(20)

phone.close()



# 2，验证服务端缓冲区取完了，又执行了recv执行，此时客户端20秒内不关闭的前提下，recv处于阻塞状态。
import socket
import time
phone = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
phone.connect(('127.0.0.1',8080))
phone.send('hello'.encode('utf-8'))
time.sleep(20)

phone.close()

# 3，验证服务端缓冲区取完了，又执行了recv执行，此时客户端处于关闭状态，则recv会取到空字符串。
import socket
import time
phone = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
phone.connect(('127.0.0.1',8080))
phone.send('hello'.encode('utf-8'))
phone.close()
```

远程执行命令的示例：

```python
import socket
import subprocess

phone = socket.socket(socket.AF_INET,socket.SOCK_STREAM)

phone.bind(('127.0.0.1',8080))

phone.listen(5)

while 1 : # 循环连接客户端
    conn, client_addr = phone.accept()
    print(client_addr)
    
    while 1:
        try:
            cmd = conn.recv(1024)
            ret = subprocess.Popen(cmd.decode('utf-8'),shell=True,stdout=subprocess.PIPE,stderr=subprocess.PIPE)
            correct_msg = ret.stdout.read()
            error_msg = ret.stderr.read()
            conn.send(correct_msg + error_msg)
        except ConnectionResetError:
            break

conn.close()
phone.close()
import socket

phone = socket.socket(socket.AF_INET,socket.SOCK_STREAM)  # 买电话

phone.connect(('127.0.0.1',8080))  # 与客户端建立连接， 拨号


while 1:
    cmd = input('>>>')
    phone.send(cmd.encode('utf-8'))
    
    from_server_data = phone.recv(1024)
    
    print(from_server_data.decode('gbk'))

phone.close()  # 挂电话
```

##### 远程执行命令

```python
import socket
import subprocess
phone = socket.socket()
phone.bind(("127.0.0.1",8848))
phone.listen(2)
while 1:
    conn,addr = phone.accept()
    print(f"连接来了:{coon,addr}")
    while 1:
        try:
            from_client_data = conn.recv(1024)
            if from_client_data.upper() == b"Q":
                print("客户端正常退出聊天了")
                break
            obj = subprocess.Pepen(from_client_data.decode('utf-8'),
                                   shell=True,
                                   stdout=subprocess.PIPE,
                                   stderr=subprocess.PIPE,)
    		result = obj.stdout.read() + obj.stderr.read()
             conn.send(result)
         except ConnectionResetError:
            print('客户端链接中断了')
            break
    conn.close()
phone.close()
import socket
phone = socket.socket()
phone.connect(("127.0.0.1",8848))
while 1:
    to_server_data = input(">>>").strip().encode('utf-8')
    if not to_server_data:
        # 服务端如果接受到了空的内容，服务端就会一直阻塞中，所以无论哪一端发送内容时，都不能为空发送
        print('发送内容不能为空')
        continue
    phone.send(to_server_data)
    if to_server_data.upper() == b'Q':
        break
    from_server_data = phone.recv(1024)  # 最多接受1024字节
    print(f'{from_server_data.decode("gbk")}')
phone.close()
```

### 粘包

![](/images/79ec95d0-52c8-4294-831d-710d95fdc6fc.png)

每个 socket 被创建后，都会分配两个缓冲区，输入缓冲区和输出缓冲区。

write()/send() 并不立即向网络中传输数据，而是先将数据写入缓冲区中，再由TCP协议将数据从缓冲区发送到目标机器。一旦将数据写入到缓冲区，函数就可以成功返回，不管它们有没有到达目标机器，也不管它们何时被发送到网络，这些都是TCP协议负责的事情。

TCP协议独立于 write()/send() 函数，数据有可能刚被写入缓冲区就发送到网络，也可能在缓冲区中不断积压，多次写入的数据被一次性发送到网络，这取决于当时的网络情况、当前线程是否空闲等诸多因素，不由程序员控制。

read()/recv() 函数也是如此，也从输入缓冲区中读取数据，而不是直接从网络中读取。

这些I/O缓冲区特性可整理如下：

1.I/O缓冲区在每个TCP套接字中单独存在； 2.I/O缓冲区在创建套接字时自动生成； 3.即使关闭套接字也会继续传送输出缓冲区中遗留的数据； 4.关闭套接字将丢失输入缓冲区中的数据。

输入输出缓冲区的默认大小一般都是 8K，可以通过 getsockopt() 函数获取：

1. unsigned optVal;
2. int optLen = sizeof(int);
3. getsockopt(servSock, SOL_SOCKET, SO_SNDBUF,(char*)&optVal, &optLen);
4. printf("Buffer length: %d\n", optVal);

socket缓冲区解释

socket缓存区的详细解释

#### 哪些情况会发生粘包

1. 接收方没有及时接收缓冲区的包，造成多个包接收（客户端发送了一段数据，服务端只收了一小部分，服务端下次再收的时候还是从缓冲区拿上次遗留的数据，产生粘包）

   ```python
   import socket
   import subprocess
   
   phone = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
   
   phone.bind(('127.0.0.1', 8080))
   
   phone.listen(5)
   
   while 1:  # 循环连接客户端
       conn, client_addr = phone.accept()
       print(client_addr)
   
       while 1:
           try:
               cmd = conn.recv(1024)
               ret = subprocess.Popen(cmd.decode('utf-8'), shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
               correct_msg = ret.stdout.read()
               error_msg = ret.stderr.read()
               conn.send(correct_msg + error_msg)
           except ConnectionResetError:
               break
   
   conn.close()
   phone.close()
   
   # 服务端
   ```

   ```python
   import socket
   
   phone = socket.socket(socket.AF_INET,socket.SOCK_STREAM)  # 买电话
   
   phone.connect(('127.0.0.1',8080))  # 与客户端建立连接， 拨号
   
   
   while 1:
       cmd = input('>>>')
       phone.send(cmd.encode('utf-8'))
   
       from_server_data = phone.recv(1024)
   
       print(from_server_data.decode('gbk'))
   
   phone.close() 
   
   # 由于客户端发的命令获取的结果大小已经超过1024，那么下次在输入命令，会继续取上次残留到缓存区的数据。
   
   客户端
   ```

2. 发送端需要等缓冲区满才发送出去，造成粘包（发送数据时间间隔很短，数据也很小，会合到一起，产生粘包）

   ```python
   import socket
   
   
   phone = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
   
   phone.bind(('127.0.0.1', 8080))
   
   phone.listen(5)
   
   conn, client_addr = phone.accept()
   
   frist_data = conn.recv(1024)
   print('1:',frist_data.decode('utf-8'))  # 1: helloworld
   second_data = conn.recv(1024)
   print('2:',second_data.decode('utf-8'))
   
   
   conn.close()
   phone.close()
   
   # 服务端
   ```

   ```python
   import socket
   
   phone = socket.socket(socket.AF_INET, socket.SOCK_STREAM)  
   
   phone.connect(('127.0.0.1', 8080)) 
   
   phone.send(b'hello')
   phone.send(b'world')
   
   phone.close()  
   
   # 两次返送信息时间间隔太短，数据小，造成服务端一次收取
   
   # 客户端
   ```

#### 粘包的解决方案

**先介绍一下struct模块：**

该模块可以把一个类型，如数字，转成固定长度的bytes

![img](https://uploadfiles.nowcoder.com/images/20200914/877190903_1600072166956_2662C697DDE960B32656C50BFB566938)

```python
import struct
# 将一个数字转化成等长度的bytes类型。
ret = struct.pack('i', 183346)
print(ret, type(ret), len(ret))

# 通过unpack反解回来
ret1 = struct.unpack('i',ret)[0]
print(ret1, type(ret1), len(ret1))


# 但是通过struct 处理不能处理太大

ret = struct.pack('l', 4323241232132324)
print(ret, type(ret), len(ret))  # 报错
```

1. low版

   - 问题的根源在于，接收端不知道发送端将要传送的字节流的长度，所以解决粘包的方法就是围绕，如何让发送端在发送数据前，把自己将要发送的字节流总数按照固定字节发送给接收端后面跟上总数据，然后接收端先接收固定字节的总字节流，再来一个死循环接收完所有数据。

   ```python
   import socket
   import subprocess
   import struct
   phone = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
   
   phone.bind(('127.0.0.1', 8080))
   
   phone.listen(5)
   
   while 1:
       conn, client_addr = phone.accept()
       print(client_addr)
   
       while 1:
           try:
               cmd = conn.recv(1024)
               ret = subprocess.Popen(cmd.decode('utf-8'), shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
               correct_msg = ret.stdout.read()
               error_msg = ret.stderr.read()
   
               # 1 制作固定报头
               total_size = len(correct_msg) + len(error_msg)
               header = struct.pack('i', total_size)
   
               # 2 发送报头
               conn.send(header)
   
               # 发送真实数据：
               conn.send(correct_msg)
               conn.send(error_msg)
           except ConnectionResetError:
               break
   
   conn.close()
   phone.close()
   
   
   # 但是low版本有问题：
   # 1，报头不只有总数据大小，而是还应该有MD5数据，文件名等等一些数据。
   # 2，通过struct模块直接数据处理，不能处理太大。
   
   # 服务端
   ```

   ```python
   import socket
   import struct
   phone = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
   
   phone.connect(('127.0.0.1',8080))
   
   
   while 1:
       cmd = input('>>>').strip()
       if not cmd: continue
       phone.send(cmd.encode('utf-8'))
   
       # 1，接收固定报头
       header = phone.recv(4)
   
       # 2，解析报头
       total_size = struct.unpack('i', header)[0]
   
       # 3，根据报头信息，接收真实数据
       recv_size = 0
       res = b''
   
       while recv_size < total_size:
   
           recv_data = phone.recv(1024)
           res += recv_data
           recv_size += len(recv_data)
   
       print(res.decode('gbk'))
   
   phone.close()
   
   # 客户端
   ```

2. 可自定制报头版

   - 整个流程的大致解释： 我们可以把报头做成字典，字典里包含将要发送的真实数据的描述信息(大小啊之类的)，然后json序列化，然后用struck将序列化后的数据长度打包成4个字节。 我们在网络上传输的所有数据 都叫做数据包，数据包里的所有数据都叫做报文，报文里面不止有你的数据，还有ip地址、mac地址、端口号等等，其实所有的报文都有报头，这个报头是协议规定的，看一下

     发送时： 先发报头长度 再编码报头内容然后发送 最后发真实内容

     接收时： 先手报头长度，用struct取出来 根据取出的长度收取报头内容，然后解码，反序列化 从反序列化的结果中取出待取数据的描述信息，然后去取真实的数据内容

     整体的流程解释

   ```python
   import socket
   import subprocess
   import struct
   import json
   phone = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
   
   phone.bind(('127.0.0.1', 8080))
   
   phone.listen(5)
   
   while 1:
       conn, client_addr = phone.accept()
       print(client_addr)
   
       while 1:
           try:
               cmd = conn.recv(1024)
               ret = subprocess.Popen(cmd.decode('utf-8'), shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
               correct_msg = ret.stdout.read()
               error_msg = ret.stderr.read()
   
               # 1 制作固定报头
               total_size = len(correct_msg) + len(error_msg)
   
               header_dict = {
                   'md5': 'fdsaf2143254f',
                   'file_name': 'f1.txt',
                   'total_size':total_size,
               }
   
               header_dict_json = json.dumps(header_dict) # str
               bytes_headers = header_dict_json.encode('utf-8')
   
               header_size = len(bytes_headers)
   
               header = struct.pack('i', header_size)
   
               # 2 发送报头长度
               conn.send(header)
   
               # 3 发送报头
               conn.send(bytes_headers)
   
               # 4 发送真实数据：
               conn.send(correct_msg)
               conn.send(error_msg)
           except ConnectionResetError:
               break
   
   conn.close()
   phone.close()
   
   # 服务端
   ```

   ```python
   import socket
   import struct
   import json
   phone = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
   
   phone.connect(('127.0.0.1',8080))
   
   
   while 1:
       cmd = input('>>>').strip()
       if not cmd: continue
       phone.send(cmd.encode('utf-8'))
   
       # 1，接收固定报头
       header_size = struct.unpack('i', phone.recv(4))[0]
   
       # 2，解析报头长度
       header_bytes = phone.recv(header_size)
   
       header_dict = json.loads(header_bytes.decode('utf-8'))
   
       # 3,收取报头
       total_size = header_dict['total_size']
   
       # 3，根据报头信息，接收真实数据
       recv_size = 0
       res = b''
   
       while recv_size < total_size:
   
           recv_data = phone.recv(1024)
           res += recv_data
           recv_size += len(recv_data)
   
       print(res.decode('gbk'))
   
   phone.close()
   
   # 客户端
   ```

### udp下的socket

- udp是无链接的，先启动哪一端都不会报错

![](/images/a34bf51d-12ff-42e7-92e2-eeca47d399a9.png)

UDP下的socket通讯流程

先从服务器端说起。服务器端先初始化Socket，然后与端口绑定(bind)，recvform接收消息，这个消息有两项，消息内容和对方客户端的地址，然后回复消息时也要带着你收到的这个客户端的地址，发送回去，最后关闭连接，一次交互结束

上代码感受一下，需要创建两个文件，文件名称随便起，为了方便看，我的两个文件名称为udp_server.py(服务端)和udp_client.py(客户端)，将下面的server端的代码拷贝到udp_server.py文件中，将下面cliet端的代码拷贝到udp_client.py的文件中，然后先运行udp_server.py文件中的代码，再运行udp_client.py文件中的代码，然后在pycharm下面的输出窗口看一下效果。

```
import socket
udp_sk = socket.socket(type=socket.SOCK_DGRAM)   #创建一个服务器的套接字
udp_sk.bind(('127.0.0.1',9000))        #绑定服务器套接字
msg,addr = udp_sk.recvfrom(1024)
print(msg)
udp_sk.sendto(b'hi',addr)                 # 对话(接收与发送)
udp_sk.close()                         # 关闭服务器套接字

# server端
import socket
ip_port=('127.0.0.1',9000)
udp_sk=socket.socket(type=socket.SOCK_DGRAM)
udp_sk.sendto(b'hello',ip_port)
back_msg,addr=udp_sk.recvfrom(1024)
print(back_msg.decode('utf-8'),addr)

# client端
```

- 类似于qq聊天的代码示例：

```python
#_*_coding:utf-8_*_
import socket
ip_port=('127.0.0.1',8081)
udp_server_sock=socket.socket(socket.AF_INET,socket.SOCK_DGRAM) #DGRAM:datagram 数据报文的意思，象征着UDP协议的通信方式
udp_server_sock.bind(ip_port)#你对外提供服务的端口就是这一个，所有的客户端都是通过这个端口和你进行通信的

while True:
    qq_msg,addr=udp_server_sock.recvfrom(1024)# 阻塞状态，等待接收消息
    print('来自[%s:%s]的一条消息:\033[1;44m%s\033[0m' %(addr[0],addr[1],qq_msg.decode('utf-8')))
    back_msg=input('回复消息: ').strip()

    udp_server_sock.sendto(back_msg.encode('utf-8'),addr)

#server端
import socket
BUFSIZE=1024
udp_client_socket=socket.socket(socket.AF_INET,socket.SOCK_DGRAM)

qq_name_dic={
    'taibai':('127.0.0.1',8081),
    'Jedan':('127.0.0.1',8081),
    'Jack':('127.0.0.1',8081),
    'John':('127.0.0.1',8081),
}


while True:
    qq_name=input('请选择聊天对象: ').strip()
    while True:
        msg=input('请输入消息,回车发送,输入q结束和他的聊天: ').strip()
        if msg == 'q':break
        if not msg or not qq_name or qq_name not in qq_name_dic:continue
        udp_client_socket.sendto(msg.encode('utf-8'),qq_name_dic[qq_name])# 必须带着自己的地址，这就是UDP不一样的地方，不需要建立连接，但是要带着自己的地址给服务端，否则服务端无法判断是谁给我发的消息，并且不知道该把消息回复到什么地方，因为我们之间没有建立连接通道

        back_msg,addr=udp_client_socket.recvfrom(BUFSIZE)# 同样也是阻塞状态，等待接收消息
        print('来自[%s:%s]的一条消息:\033[1;44m%s\033[0m' %(addr[0],addr[1],back_msg.decode('utf-8')))

udp_client_socket.close()

# client端
```