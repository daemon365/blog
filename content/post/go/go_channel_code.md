---
title: "Go channel 原理"
date: 2024-12-22T19:14:55+08:00
lastmod: 2024-12-22T19:14:55+08:00

categories:
  - go
tags:
  - go
---

## 作用

1. Go 语言的 channel 是一种 goroutine 之间的通信方式，它可以用来传递数据，也可以用来同步 goroutine 的执行。
2. chan 是 goroutine 之间的通信桥梁，可以安全地在多个 goroutine 中共享数据。
3. 使用 chan 实现 goroutine 之间的协作与同步，可用于信号传递、任务完成通知等。
4. select 配合 chan，可以同时监听多个 channel，处理任意一个可用 channel 的数据。


## 结构

```go
type hchan struct {
	qcount   uint           // 队列中的元素个数
	dataqsiz uint           // 环形队列的容量
	buf      unsafe.Pointer // 环形队列的指针
	elemsize uint16        // 元素的大小
	closed   uint32         // 是否关闭 如果以关闭则不是0
	timer    *timer // 为此 channel 提供时间控制的计时器
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

    success bool      // 通信是否成功（接收到值或因 channel 关闭被唤醒）

    waiters uint16    // 等待者数量，仅在列表头部有意义

    parent   *sudog   // 指向父节点的指针，在二叉树结构中使用
    waitlink *sudog   // g的等待链表或semaRoot
    waittail *sudog   // semaRoot的尾部
    c        *hchan   // 指向sudog所等待的 channel 
}
```

## 创建

创建一个 channel:

```go
func makechan(t *chantype, size int) *hchan {
  // 元素类型
	elem := t.Elem

	// 检查大小是否合法
	if elem.Size_ >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
  // 是否满足对齐要求
	if hchanSize%maxAlign != 0 || elem.Align_ > maxAlign {
		throw("makechan: bad alignment")
	}
  // 计算内存分配所需大小：`元素大小 * 数量`。
	mem, overflow := math.MulUintptr(elem.Size_, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	var c *hchan
	switch {
	case mem == 0:
		// 队列大小为0 说明是无缓冲的channel 直接分配hchan
    // 分配内存 hchanSize 是 hchan 结构体的大小
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		c.buf = c.raceaddr()
	case !elem.Pointers():
		// 如果元素中不包含指针 则使用一个连续的内存块 结构体和 buf 是连续的
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// 如果元素中包含指针 则使用两个内存块
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.Size_)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	lockInit(&c.lock, lockRankHchan)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.Size_, "; dataqsiz=", size, "\n")
	}
	return c
}
```

```go
const hchanSize = unsafe.Sizeof(hchan{}) + uintptr(-int(unsafe.Sizeof(hchan{}))&(maxAlign-1))

func (c *hchan) raceaddr() unsafe.Pointer {
  // 将对 channel 的读写操作视为发生在这个地址。
  // 避免使用 `qcount` 或 `dataqsiz` 的地址，
  // 因为内建函数 `len()` 和 `cap()` 会读取这些地址，
  // 而我们不希望这些操作与例如 `close()` 之类的操作发生竞争。
	return unsafe.Pointer(&c.buf)
}
```


## 写

```go
func chansend1(c *hchan, elem unsafe.Pointer) {
	chansend(c, elem, true, getcallerpc())
}

func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	if c == nil {
		if !block {
			return false
		}
    // 如果 channel 为空 挂起当前 goroutine 并报错
		gopark(nil, nil, waitReasonChanSendNilChan, traceBlockForever, 2)
		throw("unreachable")
	}

	// ......

	// 检查非阻塞模式是否可以直接返回失败结果
	if !block && c.closed == 0 && full(c) {
		return false
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)
    // 检查 channel 是否已经关闭
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

	if sg := c.recvq.dequeue(); sg != nil {
		// 如果有等待接收的 Goroutine，直接将值发送给它，跳过缓冲区
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

	if c.qcount < c.dataqsiz {
		 // 如果通道缓冲区有空间，直接将值写入缓冲区
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			racenotify(c, c.sendx, nil)
		}
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}

	if !block {
    // 非阻塞模式且无法发送值，返回 false
		unlock(&c.lock)
		return false
	}

	// 阻塞模式，当前 Goroutine 挂起等待接收者
	gp := getg()
  // 放入 acquireSudog
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}

	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)

	gp.parkingOnChan.Store(true)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceBlockChanSend, 2)
	// 确保发送值在接收者拷贝之前不会被释放
	KeepAlive(ep)

	// 唤醒后，检查状态
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	closed := !mysg.success
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
  // 回收 sudog
	releaseSudog(mysg)
	if closed {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	return true
}
```

所以阻塞写这个主要有三种模式:

1. 如果有等待接收的 Goroutine （c.recvq 里面有值），说明 buf 要么满了 要么就没有，直接将值发送给它，跳过缓冲区
2. 如果通道缓冲区有空间，直接将值写入缓冲区
3. 如果缓冲区没有空间，且是阻塞模式，当前 Goroutine 挂起等待接收者

![](/images/go_channel_write.png)

### send

```go
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	// ......

  // 如果接收者有一个有效的元素指针，则将发送者的数据直接拷贝给接收者
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	sg.success = true
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}

   // 唤醒接收者 Goroutine
	goready(gp, skip+1)
}

```

```go
func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
	// 内存拷贝
	dst := sg.elem
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.Size_)
	memmove(dst, src, t.Size_)
}
```

大致的逻辑为，取出 goroutine 然后把接受的值 COPY 到接受者的内存中，然后唤醒接受者 goroutine。
接受的内存可能是堆也可能是栈，堆还好说，如果是栈，就是在一个栈内直接操作其他的栈了，按理来说，这是不安全的。但是，这是 runtime, 我们已经把 goroutine GoPark 了，保证了它不会执行，所以这里是安全的。当然我们自己写代码时，肯定是不能这么做的。


### chanbuf && typedmemmove 

```go
func chanbuf(c *hchan, i uint) unsafe.Pointer {
  // 在 buf 上加上 i * elemsize 的偏移量
	return add(c.buf, uintptr(i)*uintptr(c.elemsize))
}

func typedmemmove(typ *abi.Type, dst, src unsafe.Pointer) {
	if dst == src {
		return
	}
	if writeBarrier.enabled && typ.Pointers() {
		// 如果写屏障启用且类型包含指针，则需要处理写屏障。
		bulkBarrierPreWrite(uintptr(dst), uintptr(src), typ.PtrBytes, typ)
	}
	// 执行内存拷贝
	memmove(dst, src, typ.Size_)
	if goexperiment.CgoCheck2 {
		cgoCheckMemmove2(typ, dst, src, 0, typ.Size_)
	}
}
```
### acquireSudog & releaseSudog

```GO
func acquireSudog() *sudog {
	mp := acquirem()
	pp := mp.p.ptr()

  // 如果 sudog 缓存为空，需要补充缓存
	if len(pp.sudogcache) == 0 {
		lock(&sched.sudoglock)
		for len(pp.sudogcache) < cap(pp.sudogcache)/2 && sched.sudogcache != nil {
			s := sched.sudogcache
			sched.sudogcache = s.next
			s.next = nil
			pp.sudogcache = append(pp.sudogcache, s)
		}
		unlock(&sched.sudoglock)
		if len(pp.sudogcache) == 0 {
			pp.sudogcache = append(pp.sudogcache, new(sudog))
		}
	}

  // 从 P 的缓存中取出一个 sudog
	n := len(pp.sudogcache)
	s := pp.sudogcache[n-1]
	pp.sudogcache[n-1] = nil
	pp.sudogcache = pp.sudogcache[:n-1]
	if s.elem != nil {
		throw("acquireSudog: found s.elem != nil in cache")
	}
	releasem(mp)
	return s
}

func releaseSudog(s *sudog) {
	// ......
  gp := getg()

	mp := acquirem() 
	pp := mp.p.ptr()
	if len(pp.sudogcache) == cap(pp.sudogcache) {
	 // 如果本地缓存已满，将部分 sudog 转移到全局缓存
		var first, last *sudog
		for len(pp.sudogcache) > cap(pp.sudogcache)/2 {
			n := len(pp.sudogcache)
			p := pp.sudogcache[n-1]
			pp.sudogcache[n-1] = nil
			pp.sudogcache = pp.sudogcache[:n-1]
			if first == nil {
				first = p
			} else {
				last.next = p
			}
			last = p
		}
		lock(&sched.sudoglock)
		last.next = sched.sudogcache
		sched.sudogcache = first
		unlock(&sched.sudoglock)
	}
  // 将 sudog 放回本地缓存
	pp.sudogcache = append(pp.sudogcache, s)
	releasem(mp)
}
```

## 读

```go
func chanrecv1(c *hchan, elem unsafe.Pointer) {
	chanrecv(c, elem, true)
}

// 带 ok 时 比如 `v, ok := <-ch`
func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
	_, received = chanrecv(c, elem, true)
	return
}

func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// ......

	// 非阻塞模式下检查失败条件
	if !block && empty(c) {
		if atomic.Load(&c.closed) == 0 {
      		// 已经没关闭 直接返回 因为这是非阻塞模式而且 buf 为空的情况
			return
		}
		if empty(c) {
			if raceenabled {
				raceacquire(c.raceaddr())
			}
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)

	if c.closed != 0 {
    	// 通道已关闭 检查是否有数据
		if c.qcount == 0 {
			if raceenabled {
				raceacquire(c.raceaddr())
			}
			unlock(&c.lock)
			if ep != nil {
        // 把数据清零 因为通道已经关闭了 
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
	} else {
		// 通道未关闭，检查是否有等待发送的 Goroutine
		if sg := c.sendq.dequeue(); sg != nil {
			// 直接从发送队列中取出值
			recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
			return true, true
		}
	}

  	// 如果缓冲区中有数据，从缓冲区接收
	if c.qcount > 0 {
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			racenotify(c, c.recvx, nil)
		}
		if ep != nil {
      		// 直接 COPY 内存
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

  	// 如果是非阻塞接收，直接返回
	if !block {
		unlock(&c.lock)
		return false, false
	}

	// 没有可用的发送方：阻塞在该通道上
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg

	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	c.recvq.enqueue(mysg)
	if c.timer != nil {
		blockTimerChan(c)
	}

	gp.parkingOnChan.Store(true)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceBlockChanRecv, 2)

	// someone woke us up
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	if c.timer != nil {
		unblockTimerChan(c)
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	success := mysg.success
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, success
}
```

所以读数据（阻塞读）的逻辑为:

1. 检查 channel 是否已经关闭 如果关闭了 而且没有数据了 直接返回
2. 如果有等待发送的 Goroutine （c.sendq 里面有值），如果无缓冲chan 直接从goroutine中取值 负责从 buf 取出值 并把数据加入末尾
3. 如果缓冲区中有数据，从缓冲区接收
4. 如果缓冲区没有数据了 挂起 goroutine 并加入 recvq 等待接收者

![](/images/go_channel_read.png)

### recv

```go
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if c.dataqsiz == 0 {
		
		if ep != nil {
			// 如果是无缓冲通道，直接 Copy 数据
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
		// 否则，通道是有缓冲通道。
        // 从队列的头部获取数据，同时通知发送方将其数据放到尾部
		qp := chanbuf(c, c.recvx)
		
		// 从队列复制数据到接收方
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		// 将发送者的数据复制到队列中
		typedmemmove(c.elemtype, qp, sg.elem)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
	}
	sg.elem = nil
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	sg.success = true
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)
}

```

## 关闭

```go
func closechan(c *hchan) {
  // 空值检查
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)
  // 如果已经关闭了 报错 不能关闭已经关闭的 channel
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	if raceenabled {
		callerpc := getcallerpc()
		racewritepc(c.raceaddr(), callerpc, abi.FuncPCABIInternal(closechan))
		racerelease(c.raceaddr())
	}

  // 设置状态
	c.closed = 1

   // 创建一个 G 列表，用于保存需要唤醒的 Goroutine
	var glist gList

	// 释放所有的读方
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}

	// 释放所有的写方 会 panic 因为向已经关闭的 channel 写数据是不允许的
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}
	unlock(&c.lock)

	// 唤醒所有的 Goroutine
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}
```

## 非阻塞读写

非阻塞的方式一般用在 `select` 中。

```go
func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
	return chansend(c, elem, false, getcallerpc())
}

func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected, received bool) {
	return chanrecv(c, elem, false)
}
```

- 在阻塞下，需要当前 goroutine 挂起时，非阻塞则不需要，直接返回 flase。
- 如果能直接读数据，则返回 true。

## select

```go
func walkSelectCases(cases []*ir.CommClause) []ir.Node {
	// ......
	switch n.Op() {
	default:
		base.Fatalf("select %v", n.Op())

	case ir.OSEND:
		// if selectnbsend(c, v) { body } else { default body }
		n := n.(*ir.SendStmt)
		ch := n.Chan
		cond = mkcall1(chanfn("selectnbsend", 2, ch.Type()), types.Types[types.TBOOL], r.PtrInit(), ch, n.Value)

	case ir.OSELRECV2:
		n := n.(*ir.AssignListStmt)
		recv := n.Rhs[0].(*ir.UnaryExpr)
		ch := recv.X
		elem := n.Lhs[0]
		if ir.IsBlank(elem) {
			elem = typecheck.NodNil()
		}
		cond = typecheck.TempAt(base.Pos, ir.CurFunc, types.Types[types.TBOOL])
		fn := chanfn("selectnbrecv", 2, ch.Type())
		call := mkcall1(fn, fn.Type().ResultsTuple(), r.PtrInit(), elem, ch)
		as := ir.NewAssignListStmt(r.Pos(), ir.OAS2, []ir.Node{cond, n.Lhs[1]}, []ir.Node{call})
		r.PtrInit().Append(typecheck.Stmt(as))
	}

	// ......
}
```

```go
func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
	return chansend(c, elem, false, getcallerpc())
}

func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected, received bool) {
	return chanrecv(c, elem, false)
}
```

改写后就是调用 `selectnbsend` 非阻塞的从 channel 发送数据，如果成功则返回 true，否则返回 false。失败了就从下个 case 继续执行。