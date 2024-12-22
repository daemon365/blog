---
title: "Go channel 原理"
date: 2024-12-22T19:14:55+08:00
lastmod: 2024-12-22T19:14:55+08:00

draft: true

categories:
  - go
tags:
  - go
---

## 结构

```go
type hchan struct {
	qcount   uint           // 队列中的元素个数
	dataqsiz uint           // 环形队列的容量
	buf      unsafe.Pointer // 环形队列的指针
	elemsize uint16        // 元素的大小
	closed   uint32         // 是否关闭 如果以关闭则不是0
	timer    *timer // 为此通道提供时间控制的计时器
	elemtype *_type // 元素的类型
	sendx    uint   // 发送索引，指示下一个发送操作的位置
	recvx    uint   // 接收索引，指示下一个接收操作的位置
	recvq    waitq  // 等待接收的等待队列
	sendq    waitq  // 等待发送的等待队列

	// 锁
	lock mutex
}
```

**waitq**

```go
type waitq struct {
	first *sudog // 首指针
	last  *sudog // 尾指针
}
```

**sudog**

```go
type sudog struct {
    g *g              // goroutine

    next *sudog       // 指向下一个sudog，用于形成链表
    prev *sudog       // 指向上一个sudog，用于形成链表
    elem unsafe.Pointer // 指向数据元素的指针（可能指向栈上的数据）

    acquiretime int64 // 获取资源的时间
    releasetime int64 // 释放资源的时间
    ticket      uint32 // 票据号码，用于排序和公平性

    isSelect bool     // 标志是否在select操作中使用此sudog

    success bool      // 通信是否成功（接收到值或因通道关闭被唤醒）

    waiters uint16    // 等待者数量，仅在列表头部有意义

    parent   *sudog   // 指向父节点的指针，在二叉树结构中使用
    waitlink *sudog   // g的等待链表或semaRoot
    waittail *sudog   // semaRoot的尾部
    c        *hchan   // 指向sudog所等待的通道
}
```