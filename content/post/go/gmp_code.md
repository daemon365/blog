---
title: "从源码分析 GMP 调度原理"
date: 2024-12-07T14:23:31+08:00
lastmod: 2024-12-07T14:23:31+08:00
showToc: true
categories:
  - go
---

**本身涉及到的 go 代码 都是基于 go 1.23.0 版本**

## 传统 OS 线程

线程是 CPU 的最小调度单位，CPU 通过不断切换线程来实现多任务的并发。这会引发一些问题（对于用户角度）：

1. 线程的创建和销毁等是昂贵的，因为要不断在用户空间和内核空间切换。
2. 线程的调度是由操作系统负责的，用户无法控制。而操作系统又可能不知道线程已经 IO 阻塞，导致线程被调度，浪费 CPU 资源。
3. 线程的栈是很大的，最新版 linux 默认是 8M，会引起内存浪费。
4. ......

所以，最简单的办法就是复用线程，go 中使用的是 M:N 模型，即 M 个 OS 线程对应 N 个 任务。

## GMP 模型

1. G

goroutine, 一个 goroutine 代表一个任务。它有自己的栈空间，默认是 2K，栈空间可以动态增长。方式就是把旧的栈空间复制到新的栈空间，然后释放旧的栈空间。它的栈是在 heap （对于 OS） 上分配的。

2. M

machine, 一个 M 代表一个 OS 线程。

3. P

processor, 一个 P 代表一个逻辑处理器，它维护了一个 goroutine 队列。P 会把 goroutine 分配给 M，M 会执行 goroutine。默认的大小为 CPU 核心数。

![](/images/gmp.png)

## 数据结构

### G

结构体在 `src/runtime/runtime2.go` 中定义，主要介绍一些重要的字段：

```go
type g struct {
  // goroutine 的栈 两个地址，分别是栈的起始地址和结束地址
  stack       stack
  // 绑定的m
  m         *m 
  // goroutine 被调度走保存的中间状态
  sched     gobuf
  // goroutine 的状态
  atomicstatus atomic.Uint32
}

type gobuf struct {
	sp   uintptr // stack pointer 栈指针
	pc   uintptr // program counter 程序要从哪里开始执行
	g    guintptr // goroutine 的 指针
	ctxt unsafe.Pointer // 保存的上下文
	ret  uintptr // 返回地址
	lr   uintptr // link register
	bp   uintptr // base pointer 栈的基地址
}
```
#### goroutine 状态

```go
// defined constants
const (
	// 未初始化
	_Gidle = iota // 0

	// 准备好了 可以被 P 调度
	_Grunnable // 1

	// 正在执行中
	_Grunning // 2

	// 正在执行系统调用
	_Gsyscall // 3

	// 正在等待 例如 channel network 等
	_Gwaiting // 4

	// 没有被使用 为了兼容性
	_Gmoribund_unused // 5

	// 未使用的 goroutine 
  // 1. 可能初始化了但是没有被使用 
  // 2. 因为会复用未扩栈的 goroutine 所以也可能上次使用完了 还没继续使用
	_Gdead // 6

	// 没有被使用 为了兼容性
	_Genqueue_unused // 7

	// 栈扩容中
	_Gcopystack // 8

	// 被抢占了 等待到 _Gwaiting 
	_Gpreempted // 9

	// 用于 GC 扫描
	_Gscan          = 0x1000
	_Gscanrunnable  = _Gscan + _Grunnable  // 0x1001
	_Gscanrunning   = _Gscan + _Grunning   // 0x1002
	_Gscansyscall   = _Gscan + _Gsyscall   // 0x1003
	_Gscanwaiting   = _Gscan + _Gwaiting   // 0x1004
	_Gscanpreempted = _Gscan + _Gpreempted // 0x1009
)
```

状态流转:

![](/images/gmp_g_status.png.png)

1. 如果 groutine 还未初始化，那么状态是 `_Gidle`
2. 初始化完毕是 `_Gdead`
3. 当被调用 go func() 时，状态变为 `_Grunnable`
4. 当被调度到 M 上执行时，状态变为 `_Grunning`
5. 执行完毕后，状态变为 `_Gdead`
6. 如果 goroutine 阻塞，状态变为 `_Gwaiting` 等待阻塞完毕 状态再变为 `_Grunnable` 等待调度
7. 如果 goroutine 被抢占 （gc 要 STW 时），状态变为 `_Gpreempted` 等待变成 `_Gwaiting` 
8. 如果发生系统调用，状态变为 `_Gsyscall` 如果很快完成（10ms） 状态会变为 `_Grunning` 继续执行 否则会变为 `_Grunnable` 等待调度
9. 如果发生栈扩容，状态变为 `_Gcopystack` 等待栈扩容完毕 状态变为 `_Grunnable` 等待调度

### M

结构体在 `src/runtime/runtime2.go` 中定义，主要介绍一些重要的字段：

```go
type m struct {
  g0      *g  
  // 寄存器上下文
  morebuf gobuf
  // tls 是线程本地存储 用于存储 M 相关的线程本地数据 包括当前 G 的引用等重要信息
  tls           [tlsSlots]uintptr
  // 现在正在执行的 goroutine
  curg          *g 

  // 1. 正常执行： p 有效
  // 2. 系统调用前： p -> oldp
  // 3. 系统调用中： p == nil
  // 4. 系统调用返回： 尝试重新获取 oldp
	p             puintptr 
	nextp         puintptr
	oldp          puintptr
}
```

g0： 一个特殊的 g 用于执行调度任务 它未使用 go runtime 的 stack 而是使用 os stack
流程大概为用户态的 g -> g0 调度 -> 用户的其他 g

### P

结构体在 `src/runtime/runtime2.go` 中定义，主要介绍一些重要的字段：

```go
type p struct {
	// p 的状态
	status      uint32 
  // 分配内存使用 每个p 都有的目的是少加锁
  mcache      *mcache
  // 定长的 queue 用于存储 goroutine
  runqhead uint32
	runqtail uint32
	runq     [256]guintptr
  //  下个运行的 goroutine 主要用来快速调度 比如从 chan 读取数据，把 g 放到 runnext 中 当完成读取时 直接从 runnext 中取出来执行
  runnext guintptr

}
```

状态：

```go
const (
	// 空闲
	_Pidle = iota

	// 正在运行中
	_Prunning

	// 正在执行系统调用
	_Psyscall

	// GC 停止
	_Pgcstop

	// 死亡状态
	_Pdead
)
```

## 调度

go 有三种进行到调度的方式：

1. 用户 goroutine 主动执行 runtime.Gosched() 会把当前 goroutine 放到队列中等待调度
2. 用户 goroutine 阻塞，例如 channel 读写，网络 IO 等 会主动调用修改自己状态并切换到 g0 执行调度任务
3. go runtime 中有个 OS 线程 （名称是 sysmon） 检测到 goroutine 超时（上次执行到现在超过 10ms）那就会给线程发信号 使其切换到 g0 执行调度任务

***为什么 sysmon 使用物理线程而不是 goroutine 呢？***

因为所有 p 上正在执行的 g 都阻塞住了 比如 `for {}` 那么其他的 g 永远无法执行了包括负责检测的 sysmon


### 主动调度

```go
func Gosched() {
	checkTimeouts()
	mcall(gosched_m)
}
```

### 阻塞调度

```go
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceReason traceBlockReason, traceskip int) {
	if reason != waitReasonSleep {
		checkTimeouts() // timeouts may expire while two goroutines keep the scheduler busy
	}
	mp := acquirem()
	gp := mp.curg
	status := readgstatus(gp)
	if status != _Grunning && status != _Gscanrunning {
		throw("gopark: bad g status")
	}
	mp.waitlock = lock
	mp.waitunlockf = unlockf
	gp.waitreason = reason
	mp.waitTraceBlockReason = traceReason
	mp.waitTraceSkip = traceskip
	releasem(mp)
	// can't do anything that might move the G between Ms here.
	mcall(park_m)
}
```

### 抢占调度

```go
// m 在 start 的时候会注册一些信号处理函数
func initsig(preinit bool) {
	for i := uint32(0); i < _NSIG; i++ {
		// ...
		setsig(i, abi.FuncPCABIInternal(sighandler))
	}
}

// sighandler -> doSigPreempt -> asyncPreempt （去汇编代码里找） -> asyncPreempt2 
func asyncPreempt2() {
	gp := getg()
	gp.asyncSafePoint = true
	if gp.preemptStop {
		mcall(preemptPark)
	} else {
		mcall(gopreempt_m)
	}
	gp.asyncSafePoint = false
}


// sysmon 发信号 
// sysmon -> retake -> preemptone -> preemptM
func preemptM(mp *m) {
	if mp.signalPending.CompareAndSwap(0, 1) {
		// ...
		signalM(mp, sigPreempt)
	}
}

func signalM(mp *m, sig int) {
	tgkill(getpid(), int(mp.procid), sig)
}

// 代码在在汇编里 就是对线程发送信号 系统调用
func tgkill(tgid, tid, sig int)
```

### 调度代码

可以看到调度代码都是通过 mcall 调用的，mcall 会切换到 g0 执行调度任务 如果参数的函数不太一样 但是都是处理一些状态信息等，最好都会执行到 schedule 函数。


```go
func schedule() {
	// 核心代码就是选一个 g 去执行
	gp, inheritTime, tryWakeP := findRunnable() // blocks until work is available

	execute(gp, inheritTime)
}
```

findRunnable：

```go
func findRunnable() (gp *g, inheritTime, tryWakeP bool) {
	
	// Try to schedule a GC worker.
	if gcBlackenEnabled != 0 {
		gp, tnow := gcController.findRunnableGCWorker(pp, now)
		if gp != nil {
			return gp, false, true
		}
		now = tnow
	}

	if pp.schedtick%61 == 0 && sched.runqsize > 0 {
		lock(&sched.lock)
		gp := globrunqget(pp, 1)
		unlock(&sched.lock)
		if gp != nil {
			return gp, false, false
		}
	}

	// local runq
	if gp, inheritTime := runqget(pp); gp != nil {
		return gp, inheritTime, false
	}

	// global runq
	if sched.runqsize != 0 {
		lock(&sched.lock)
		gp := globrunqget(pp, 0)
		unlock(&sched.lock)
		if gp != nil {
			return gp, false, false
		}
	}

	if netpollinited() && netpollAnyWaiters() && sched.lastpoll.Load() != 0 {
		if list, delta := netpoll(0); !list.empty() { // non-blocking
			gp := list.pop()
			injectglist(&list)
			netpollAdjustWaiters(delta)
			trace := traceAcquire()
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.ok() {
				trace.GoUnpark(gp, 0)
				traceRelease(trace)
			}
			return gp, false, false
		}
	}

	// Spinning Ms: steal work from other Ps.
	//
	// Limit the number of spinning Ms to half the number of busy Ps.
	// This is necessary to prevent excessive CPU consumption when
	// GOMAXPROCS>>1 but the program parallelism is low.
	if mp.spinning || 2*sched.nmspinning.Load() < gomaxprocs-sched.npidle.Load() {
		if !mp.spinning {
			mp.becomeSpinning()
		}

		gp, inheritTime, tnow, w, newWork := stealWork(now)
		if gp != nil {
			// Successfully stole.
			return gp, inheritTime, false
		}
		if newWork {
			// There may be new timer or GC work; restart to
			// discover.
			goto top
		}

		now = tnow
		if w != 0 && (pollUntil == 0 || w < pollUntil) {
			// Earlier timer to wait for.
			pollUntil = w
		}
	}

}
```

简化了一下代码还是很多 价绍一些这个功能吧

1. 优先执行 GC worker 
2. 每 61 次 从全局队列中获取一个 g 去执行 作用是 防止所有 p 的本地队列谁都非常多 导致全局队列的 g 饿死
3. 从本地队列中获取一个 g 去执行 有限使用 runnext 
4. 从全局队列中获取一个 g 去执行 并 load 一些到本地队列
5. 如果有网络 IO 准备好了 就从网络 IO 中获取一个 g 去执行 （go 中网络 epoll_wait 正常情况下使用的阻塞模式）
6. 从其他的 p 中偷取 g 去执行 （cas 保证数据安全）

execute：

```go
func execute(gp *g, inheritTime bool) {
	// 修改状态
	casgstatus(gp, _Grunnable, _Grunning)
  // 执行
	gogo(&gp.sched)
}
```

我的 arch 是 amd64 所以代码在 `src/runtime/asm_amd64.s` 中

```go
TEXT runtime·gogo(SB), NOSPLIT, $0-8
	MOVQ	buf+0(FP), BX	  // 将 gobuf 指针加载到 BX 寄存器
	MOVQ	gobuf_g(BX), DX  // 将 gobuf 中保存的 g 指针加载到 DX
	MOVQ	0(DX), CX	  // 检查 g 不为 nil
	JMP	gogo<>(SB)

TEXT gogo<>(SB), NOSPLIT, $0
	get_tls(CX)
	MOVQ	DX, g(CX)
	MOVQ	DX, R14		// set the g register
  // 恢复寄存器状态 （sp ret bp ctxt） 执行 
	MOVQ	gobuf_sp(BX), SP	// restore SP
	MOVQ	gobuf_ret(BX), AX
	MOVQ	gobuf_ctxt(BX), DX
	MOVQ	gobuf_bp(BX), BP
  // 加载之后 清空 go 的 gobuf 结构体 为了给 gc 节省压力
	MOVQ	$0, gobuf_sp(BX)	
	MOVQ	$0, gobuf_ret(BX)
	MOVQ	$0, gobuf_ctxt(BX)
	MOVQ	$0, gobuf_bp(BX)
  // 跳转到保存的 PC （程序执行到哪了） 去执行
	MOVQ	gobuf_pc(BX), BX
	JMP	BX
```

### syscall

我的 arch 是 and64 操作系统是 linux 所以代码在 `src/runtime/asm_linux_amd64.s` 中

```go
TEXT ·SyscallNoError(SB),NOSPLIT,$0-48
	CALL	runtime·entersyscall(SB)
	MOVQ	a1+8(FP), DI
	MOVQ	a2+16(FP), SI
	MOVQ	a3+24(FP), DX
	MOVQ	$0, R10
	MOVQ	$0, R8
	MOVQ	$0, R9
	MOVQ	trap+0(FP), AX	// syscall entry
	SYSCALL
	MOVQ	AX, r1+32(FP)
	MOVQ	DX, r2+40(FP)
	CALL	runtime·exitsyscall(SB)
	RET
```

系统调用前执行这个函数：

```go
func entersyscall() {
	fp := getcallerfp()
	reentersyscall(getcallerpc(), getcallersp(), fp)
}

func reentersyscall(pc, sp, bp uintptr) {
	// 保存寄存器信息
	save(pc, sp, bp)
	gp.syscallsp = sp
	gp.syscallpc = pc
	gp.syscallbp = bp
  // 修改 g 状态
	casgstatus(gp, _Grunning, _Gsyscall)
	

	if sched.sysmonwait.Load() {
		systemstack(entersyscall_sysmon)
		save(pc, sp, bp)
	}

	if gp.m.p.ptr().runSafePointFn != 0 {
		// runSafePointFn may stack split if run on this stack
		systemstack(runSafePointFn)
		save(pc, sp, bp)
	}

	gp.m.syscalltick = gp.m.p.ptr().syscalltick
	pp := gp.m.p.ptr()
  // 解绑 P 和 M 并设置 oldP 为当前 P 等待系统调用之后重新绑定
	pp.m = 0
	gp.m.oldp.set(pp)
	gp.m.p = 0
  // 修改 P 的状态为 syscall
	atomic.Store(&pp.status, _Psyscall)
	if sched.gcwaiting.Load() {
		systemstack(entersyscall_gcwait)
		save(pc, sp, bp)
	}

	gp.m.locks--
}
```

系统调用后执行这个函数：

```go
func exitsyscall() {
	// 如果之前保存的oldp不为空 那么重新绑定
	if exitsyscallfast(oldp) {
		// 设置状态为 runnable 并重新执行
		casgstatus(gp, _Gsyscall, _Grunning)
		if sched.disable.user && !schedEnabled(gp) {
			// Scheduling of this goroutine is disabled.
			Gosched()
		}

		return
	}
  // 切换到 g0 执行 exitsyscall0
	mcall(exitsyscall0)
}

```

```go
func exitsyscall0(gp *g) {
	// 修改 g 状态到 _Grunnable 让重新可调度
	casgstatus(gp, _Gsyscall, _Grunnable)
	
	// 删除 gm 的绑定
	dropg()
	lock(&sched.lock)
	// 找个空闲的 p （状态为 _Gidle） 与 M 绑定
  var pp *p
	if schedEnabled(gp) {
		pp, _ = pidleget(0)
	}
	var locked bool
	if pp == nil {
    // 如果绑定失败了 直接把 g 放到全局队列中
		globrunqput(gp)
		locked = gp.lockedm != 0
	} else if sched.sysmonwait.Load() {
    // 如果 sysmon 在等待 那么唤醒它
		sched.sysmonwait.Store(false)
		notewakeup(&sched.sysmonnote)
	}
	unlock(&sched.lock)
  // 如果找到 p 了 那么就去执行
	if pp != nil {
		acquirep(pp)
		execute(gp, false) // Never returns.
	}
	if locked {
		// Wait until another thread schedules gp and so m again.
		//
		// N.B. lockedm must be this M, as this g was running on this M
		// before entersyscall.
		stoplockedm()
		execute(gp, false) // Never returns.
	}
  // 如果没有 P 给我这个 M 绑定的话 那么把 M 休眠并加入到 schedlink 队列中  做复用
	stopm()
  // 直到有新的 g 被调度到这个 M 上
	schedule() // Never returns.
}

```






