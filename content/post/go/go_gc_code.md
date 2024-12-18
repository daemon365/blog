---
title: "Go 垃圾回收"
date: 2024-12-17T11:06:10+08:00

categories:
  - go
tags:
  - go
---

*** Go 代码基于 v1.23.0 ***

## 介绍

- [golang GC 垃圾回收机制](https://daemon365.dev/post/go/golang_gc_garbage_collection_mechanism/)

## 触发条件

1. 用户调用 `runtime.GC` 主动触发
2. Go 程序检测到距上次 GC 内存分配增长超过一定比例时（默认 100%）触发
3. 定时触发 （默认 2 min）

## 定时触发

代码在 `src/runtime/proc.go` 中

```go
func init() {
	go forcegchelper()
}


func forcegchelper() {
	forcegc.g = getg()
	lockInit(&forcegc.lock, lockRankForcegc)
	for {
		lock(&forcegc.lock)
		if forcegc.idle.Load() {
			throw("forcegc: phase error")
		}
		forcegc.idle.Store(true)
        // 阻塞住 goroutine 当有人解开时 开启 gc
		goparkunlock(&forcegc.lock, waitReasonForceGCIdle, traceBlockSystemGoroutine, 1)
		// this goroutine is explicitly resumed by sysmon
		if debug.gctrace > 0 {
			println("GC forced")
		}
		// goroutine 被唤醒了，调用 gcStart 开启 gc
		gcStart(gcTrigger{kind: gcTriggerTime, now: nanotime()})
	}
}
```

那么唤醒 goroutine 的逻辑代码在哪里呢？在 `sysmon` 物理线程中

```go
func sysmon() {
    // ......
    for {
        // ......

        // 使用 test 方法判断是否需要触发 gc
        if t := (gcTrigger{kind: gcTriggerTime, now: now}); t.test() && forcegc.idle.Load() {
			lock(&forcegc.lock)
			forcegc.idle.Store(false)
            // 需要 gc 时，把 g 放入 list 中 并使用 injectglist 执行唤醒操作
			var list gList
			list.push(forcegc.g)
			injectglist(&list)
			unlock(&forcegc.lock)
		}
        // ......
    }
}

func injectglist(glist *gList) {
    // ......

    // 启动指定数量的空闲M
	startIdle := func(n int) {
		for i := 0; i < n; i++ {
			mp := acquirem() // See comment in startm.
			lock(&sched.lock)

			pp, _ := pidlegetSpinning(0)
			if pp == nil {
				unlock(&sched.lock)
				releasem(mp)
				break
			}

			startm(pp, false, true)
			unlock(&sched.lock)
			releasem(mp)
		}
	}

	pp := getg().m.p.ptr()
	if pp == nil {
        // 没有 P 的情况下，直接把 glist 放入全局队列 并启动一些空闲M
		lock(&sched.lock)
		globrunqputbatch(&q, int32(qsize))
		unlock(&sched.lock)
		startIdle(qsize)
		return
	}
    // 逐个将G放入全局队列，并启动相应的M
	npidle := int(sched.npidle.Load())
	var (
		globq gQueue
		n     int
	)
	for n = 0; n < npidle && !q.empty(); n++ {
		g := q.pop()
		globq.pushBack(g)
	}
	if n > 0 {
		lock(&sched.lock)
		globrunqputbatch(&globq, int32(n))
		unlock(&sched.lock)
		startIdle(n)
		qsize -= n
	}

    // 将剩余的G放入当前P的本地队列。
	if !q.empty() {
		runqputbatch(pp, &q, qsize)
	}

	// 唤醒一个P，以防在添加G到队列后有P变为空闲状态。
	wakep()
}
```

### test

- 根据 gcTrigger 的类型检测是否应该触发垃圾收集。

```go
type gcTrigger struct {
	kind gcTriggerKind
	now  int64  // gcTriggerTime: current time
	n    uint32 // gcTriggerCycle: cycle number to start
}

type gcTriggerKind int

const (
	gcTriggerHeap gcTriggerKind = iota
	gcTriggerTime
	gcTriggerCycle
)


func (t gcTrigger) test() bool {
    // 检查是否禁用了 GC，程序是否处于 panic 状态，或 GC 阶段是否不是关闭状态。 
	if !memstats.enablegc || panicking.Load() != 0 || gcphase != _GCoff {
		return false
	}
	switch t.kind {
	case gcTriggerHeap:
        // heap 内存的方式触发
		trigger, _ := gcController.trigger()
		return gcController.heapLive.Load() >= trigger
	case gcTriggerTime:
        // 定时触发
		if gcController.gcPercent.Load() < 0 {
			return false
		}
        // 如果现在的时间减去上次 gc 的时间大于 forcegcperiod （2min） 就触发
		lastgc := int64(atomic.Load64(&memstats.last_gc_nanotime))
		return lastgc != 0 && t.now-lastgc > forcegcperiod
	case gcTriggerCycle:
		// 上次是否执行完毕
		return int32(t.n-work.cycles.Load()) > 0
	}
	return true
}

```

## Heap 增长触发

`mallocgc` 不但 malloc 了内存，还会检查是否需要触发 gc

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
    assistG := deductAssistCredit(size)

    shouldhelpgc := false
    // ......

    if size <= maxSmallSize-mallocHeaderSize  {
        // ......
        // 如果 mcache 中满了 要向上申请了 shouldhelpgc = true 后续判断
        v, span, shouldhelpgc = c.nextFree(tinySpanClass)
        // ......
    } else {
        // 大于 32k 
        shouldhelpgc = true
        // ......
    }

    // 如果开启了 GC 新对象都标黑
    if gcphase != _GCoff {
		gcmarknewobject(span, uintptr(x))
	}

    // test() 看下需不需要 gc
    if shouldhelpgc {
		if t := (gcTrigger{kind: gcTriggerHeap}); t.test() {
			gcStart(t)
		}
	}
}
```

```go
func gcmarknewobject(span *mspan, obj uintptr) {
	span.markBitsForIndex(objIndex).setMarked()
}

```

## 主动触发

```go
func GC() {
    // ......
    gcStart(gcTrigger{kind: gcTriggerCycle, n: n + 1})
    // ......
}
```


## gcStart

gcStart 为 gc 的标记主流程

```go
func gcStart(trigger gcTrigger) {
	// ......

	// 再次检查一下是不是能 GC 如果上次没清扫完，给清扫完
	for trigger.test() && sweepone() != ^uintptr(0) {
	}

	// 上锁
	semacquire(&work.startSema)
	// 加锁了 再次检查一下
	if !trigger.test() {
		semrelease(&work.startSema)
		return
	}

	// ......

    // 启动 GC 标记工作线程。
	gcBgMarkStartWorkers()

    // 重置标记
	systemstack(gcResetMarkState)
    // ......
    // Stop The World
	systemstack(func() {
		stw = stopTheWorldWithSema(stwGCSweepTerm)
	})


	// 在开始并发扫描之前完成清扫 
	systemstack(func() {
		finishsweep_m()
	})

	// ......

	// 使用多少个核心 GC
	gcController.startCycle(now, int(gomaxprocs), trigger)

	// ......

    // 进入并发标记阶段并启用写屏障。
    setGCPhase(_GCmark)

    // 标记所有活跃的 tinyalloc 块。
    gcMarkTinyAllocs()

	// 开始标记
	systemstack(func() {
        // start the world 
		now = startTheWorldWithSema(0, stw)
		work.pauseNS += now - stw.startedStopping
		work.tMark = now

		// 释放 CPU 限制器
		gcCPULimiter.finishGCTransition(now)
	})

	// 在 STW 模式下，在 Gosched() 之前释放 world sema，
    // 因为我们稍后需要再次获取它，但在这个 goroutine 变为可运行之前，我们可能会自我死锁。
	semrelease(&worldsema)
	releasem(mp)

	 // 确保在 STW 模式下阻塞，而不是返回到用户代码。
	if mode != gcBackgroundMode {
		Gosched()
	}
    // 释放开始信号量
	semrelease(&work.startSema)
}
```

### gcBgMarkStartWorkers

gcBgMarkStartWorkers 启动 P 个数个 goroutine 去做并发标记

```go
func gcBgMarkStartWorkers() {
	// ......
	
	for gcBgMarkWorkerCount < gomaxprocs {
		go gcBgMarkWorker(ready)
		releasem(mp)
		<-ready
	}

    if incnwait == work.nproc && !gcMarkWorkAvailable(nil) {
			releasem(node.m.ptr())
			node.m.set(nil)

			gcMarkDone()
		}
}
```

```go
func gcBgMarkWorker(ready chan struct{}) {
	gp := getg()
    // 把 node 组装成 node
	node := new(gcBgMarkWorkerNode)
	node.gp.set(gp)
	node.m.set(acquirem())

	ready <- struct{}{}
	for {
		// 阻塞当前 g 并把 node 传入 gcBgMarkWorkerPool
		gopark(func(g *g, nodep unsafe.Pointer) bool {
			node := (*gcBgMarkWorkerNode)(nodep)
			if mp := node.m.ptr(); mp != nil {
				releasem(mp)
			}
			gcBgMarkWorkerPool.push(&node.node)
			return true
		}, unsafe.Pointer(node), waitReasonGCWorkerIdle, traceBlockSystemGoroutine, 0)

		// ......

        // 设置 gc goroutine 使用资源的方式
		systemstack(func() {
			casGToWaitingForGC(gp, _Grunning, waitReasonGCWorkerActive)
			// gcDrain(......)
            // .....
			casgstatus(gp, _Gwaiting, _Grunning)
		})

		// ......
		
	}
}
```

### stopTheWorldWithSema

停止所有用户的 goroutine 做一些事情

```go
func stopTheWorldWithSema(reason stwReason) worldStop {
	
    // 加锁 并抢占所有的 goroutine
	lock(&sched.lock)
	preemptall()
	// 将所有 P 设置为 _Pgcstop 状态 （当前 运行中 和空闲）
	gp.m.p.ptr().status = _Pgcstop 
	gp.m.p.ptr().gcStopTime = start
	sched.stopwait--
	trace = traceAcquire()
	for _, pp := range allp {
		s := pp.status
		if s == _Psyscall && atomic.Cas(&pp.status, s, _Pgcstop) {
			if trace.ok() {
				trace.ProcSteal(pp, false)
			}
			pp.syscalltick++
			pp.gcStopTime = nanotime()
			sched.stopwait--
		}
	}
	if trace.ok() {
		traceRelease(trace)
	}
	now := nanotime()
	for {
		pp, _ := pidleget(now)
		if pp == nil {
			break
		}
		pp.status = _Pgcstop
		pp.gcStopTime = nanotime()
		sched.stopwait--
	}
	wait := sched.stopwait > 0
	unlock(&sched.lock)

	// 当前 P 不能被抢占 等待完成
	if wait {
		for {
			if notetsleep(&sched.stopnote, 100*1000) {
				noteclear(&sched.stopnote)
				break
			}
			preemptall()
		}
	}

	// ......

	worldStopped()

	return worldStop{
		reason:           reason,
		startedStopping:  start,
		finishedStopping: finish,
		stoppingCPUTime:  stoppingCPUTime,
	}
}
```


### startCycle

设置可以使用多少个核心进行 GC

```go
func (c *gcControllerState) startCycle(markStartTime int64, procs int, trigger gcTrigger) {
	// P * 25% 然后取整
	totalUtilizationGoal := float64(procs) * gcBackgroundUtilization
	dedicatedMarkWorkersNeeded := int64(totalUtilizationGoal + 0.5)
	utilError := float64(dedicatedMarkWorkersNeeded)/totalUtilizationGoal - 1
	const maxUtilError = 0.3
    // 如果不能整除 算出时间片
	if utilError < -maxUtilError || utilError > maxUtilError {
		if float64(dedicatedMarkWorkersNeeded) > totalUtilizationGoal {
			dedicatedMarkWorkersNeeded--
		}
		c.fractionalUtilizationGoal = (totalUtilizationGoal - float64(dedicatedMarkWorkersNeeded)) / float64(procs)
	} else {
		c.fractionalUtilizationGoal = 0
	}

	if debug.gcstoptheworld > 0 {
		dedicatedMarkWorkersNeeded = int64(procs)
		c.fractionalUtilizationGoal = 0
	}


}
```

### setGCPhase

开启混合写屏障

```go
func setGCPhase(x uint32) {
	atomic.Store(&gcphase, x)
	writeBarrier.enabled = gcphase == _GCmark || gcphase == _GCmarktermination
}
```

```assembly 
TEXT gcWriteBarrier<>(SB),NOSPLIT,$112
// ...
CALL	runtime·wbBufFlush(SB)
// ...
```

```go
func wbBufFlush() {
	// 如果当前 M（操作系统线程）正在终止过程中
	if getg().m.dying > 0 {
		getg().m.p.ptr().wbBuf.discard()
		return
	}

	systemstack(func() {
		wbBufFlush1(getg().m.p.ptr())
	})
}

func wbBufFlush1(pp *p) {
	// 获取缓冲区中的所有指针
	start := uintptr(unsafe.Pointer(&pp.wbBuf.buf[0]))
	n := (pp.wbBuf.next - start) / unsafe.Sizeof(pp.wbBuf.buf[0])
	ptrs := pp.wbBuf.buf[:n]

	// Poison the buffer to make extra sure nothing is enqueued
	// while we're processing the buffer.
	pp.wbBuf.next = 0

	if useCheckmark {
		// Slow path for checkmark mode.
		for _, ptr := range ptrs {
			shade(ptr)
		}
		pp.wbBuf.reset()
		return
	}

	// 标记缓冲区中的所有指针，并且只记录被置灰的指针。
    // 使用缓冲区本身临时记录被置灰的指针。
	gcw := &pp.gcw
	pos := 0
	for _, ptr := range ptrs {
		if ptr < minLegalPointer {
			continue
		}
		obj, span, objIndex := findObject(ptr, 0, 0)
		if obj == 0 {
			continue
		}
		mbits := span.markBitsForIndex(objIndex)
		if mbits.isMarked() {
			continue
		}
		mbits.setMarked()
		arena, pageIdx, pageMask := pageIndexOf(span.base())
		if arena.pageMarks[pageIdx]&pageMask == 0 {
			atomic.Or8(&arena.pageMarks[pageIdx], pageMask)
		}

		if span.spanclass.noscan() {
			gcw.bytesMarked += uint64(span.elemsize)
			continue
		}
		ptrs[pos] = obj
		pos++
	}

	// 将被置灰的指针批量放入 gcw 中
	gcw.putBatch(ptrs[:pos])

	pp.wbBuf.reset()
}

```

### gcMarkTinyAllocs

把所有的 tinyalloc 标记为灰色

```go
func gcMarkTinyAllocs() {
	assertWorldStopped()

	for _, p := range allp {
		c := p.mcache
		if c == nil || c.tiny == 0 {
			continue
		}
		_, span, objIndex := findObject(c.tiny, 0, 0)
		gcw := &p.gcw
		greyobject(c.tiny, 0, 0, span, gcw, objIndex)
	}
}
```

### startTheWorldWithSema

```go
func startTheWorldWithSema(now int64, w worldStop) int64 {
	// ......

	worldStarted()

    // 唤醒所有的 P
	for p1 != nil {
		p := p1
		p1 = p1.link.ptr()
		if p.m != 0 {
			mp := p.m.ptr()
			p.m = 0
			if mp.nextp != 0 {
				throw("startTheWorld: inconsistent mp->nextp")
			}
			mp.nextp.set(p)
			notewakeup(&mp.park)
		} else {
			newm(nil, p, -1)
		}
	}

	// ......

	return now
}

```

## worker 标记 

在 goroutine 调度中会执行 `findRunnableGCWorker` goroutine 这个就是启动标记 worker 的函数

```go
func (c *gcControllerState) findRunnableGCWorker(pp *p, now int64) (*g, int64) {
	// ......

    // 从 gcBgMarkWorkerPool 拿出 node 
	node := (*gcBgMarkWorkerNode)(gcBgMarkWorkerPool.pop())
	// 开启
	casgstatus(gp, _Gwaiting, _Grunnable)

	return gp, now
}
```

## gcDrain

gcDrain 是标记的主流程

```go
func gcDrain(gcw *gcWork, flags gcDrainFlags) {
	// ......
	
	// 标记根对象
	if work.markrootNext < work.markrootJobs {
		for !(gp.preempt && (preemptible || sched.gcwaiting.Load() || pp.runSafePointFn != 0)) {
			job := atomic.Xadd(&work.markrootNext, +1) - 1
			if job >= work.markrootJobs {
				break
			}
			markroot(gcw, job, flushBgCredit)
			if check != nil && check() {
				goto done
			}
		}
	}

	for !(gp.preempt && (preemptible || sched.gcwaiting.Load() || pp.runSafePointFn != 0)) {
		if work.full == 0 {
			gcw.balance()
		}

        // 尝试从本地队列中获取对象
		b := gcw.tryGetFast()
		if b == 0 {
            // 如果没有从全局队列中获取 加锁
			b = gcw.tryGet()
			if b == 0 {
				// 刷新写屏障缓存
				wbBufFlush()
				b = gcw.tryGet()
			}
		}
		if b == 0 {
			break
		}
        // 清扫对象
		scanobject(b, gcw)

		// ......
	}

done:
	// 记录积分
	if gcw.heapScanWork > 0 {
		gcController.heapScanWork.Add(gcw.heapScanWork)
		if flushBgCredit {
			gcFlushBgCredit(gcw.heapScanWork - initScanWork)
		}
		gcw.heapScanWork = 0
	}
}
```

### scanobject

```go
func scanobject(b uintptr, gcw *gcWork) {
	var tp typePointers
	if n > maxObletBytes {
		// 大对象分块
		if b == s.base() {
			for oblet := b + maxObletBytes; oblet < s.base()+s.elemsize; oblet += maxObletBytes {
				if !gcw.putFast(oblet) {
					gcw.put(oblet)
				}
			}
		n = s.base() + s.elemsize - b
		n = min(n, maxObletBytes)
		tp = s.typePointersOfUnchecked(s.base())
		tp = tp.fastForward(b-tp.addr, b+n)
	} else {
		tp = s.typePointersOfUnchecked(b)
	}

	var scanSize uintptr
	for {
		var addr uintptr
		if tp, addr = tp.nextFast(); addr == 0 {
			if tp, addr = tp.next(b + n); addr == 0 {
				break
			}
		}

		scanSize = addr - b + goarch.PtrSize

		obj := *(*uintptr)(unsafe.Pointer(addr))

		if obj != 0 && obj-b >= n {
			if obj, span, objIndex := findObject(obj, b, addr-b); obj != 0 {
                // 设置为灰色
				greyobject(obj, b, addr-b, span, gcw, objIndex)
			}
		}
	}
    }
}

```

**greyobject**

gcmarkBits 是一个 bitmaps 代表了一个对象是不是可用（gc 过程中）
1. 如果 Bit = 1 对象不在 gcw 中 黑色
2. 如果 Bit = 0 对象不在 gcw 中 白色
3. 如果对象在 gcw 中 灰色

```go
func greyobject(obj, base, off uintptr, span *mspan, gcw *gcWork, objIndex uintptr) {

    mbits := span.markBitsForIndex(objIndex)
	if useCheckmark {
	} else {
		mbits.setMarked()
	}

    // 把对象放入 gcw 中
	sys.Prefetch(obj)
	if !gcw.putFast(obj) {
		gcw.put(obj)
	}
}

```

```go
func (s *mspan) markBitsForIndex(objIndex uintptr) markBits {
	bytep, mask := s.gcmarkBits.bitp(objIndex)
	return markBits{bytep, mask, objIndex}
}

func (m markBits) setMarked() {
	atomic.Or8(m.bytep, m.mask)
}
```

## 用户 goroutine 清扫


```go
func deductAssistCredit(size uintptr) *g {
	var assistG *g
	if gcBlackenEnabled != 0 {
		assistG = getg()
		if assistG.m.curg != nil {
			assistG = assistG.m.curg
		}
		assistG.gcAssistBytes -= int64(size)
        // 每个 G 都有自己的积分 当积分小于 0 启动 gc
		if assistG.gcAssistBytes < 0 {
			gcAssistAlloc(assistG)
		}
	}
	return assistG
}
```

```go
func gcAssistAlloc(gp *g) {
	// ......
retry:
	// 处理下 assistG 的积分 需不需要启动 gc

	// Perform assist work
	systemstack(func() {
		gcAssistAlloc1(gp, scanWork)
	})
    // ......
	
}
```

```go
func gcAssistAlloc1(gp *g, scanWork int64) {
	// ......
	gcw := &getg().m.p.ptr().gcw
	workDone := gcDrainN(gcw, scanWork)

	casgstatus(gp, _Gwaiting, _Grunning)
    // ......
}
```

## 终止标记 gcMarkDone

在 gcBgMarkStartWorkers 中执行 gcMarkDone 终止

```go
func gcMarkDone() {
	

top:
	// 清除写屏障的缓存和处理gcw
	gcMarkDoneFlushed = 0
	forEachP(waitReasonGCMarkTermination, func(pp *p) {
		wbBufFlush1(pp)

		pp.gcw.dispose()
		if pp.gcw.flushedWork {
			atomic.Xadd(&gcMarkDoneFlushed, 1)
			pp.gcw.flushedWork = false
		}
	})

	if gcMarkDoneFlushed != 0 {
		// 还有没处理的 跳回去再处理一次
		semrelease(&worldsema)
		goto top
	}

	// Stop The World
	systemstack(func() {
		stw = stopTheWorldWithSema(stwGCMarkTerm)
	})
	// 清楚写屏障中的缓存 如果还有没处理的 跳回去再处理一次
	restart := false
	systemstack(func() {
		for _, p := range allp {
			wbBufFlush1(p)
			if !p.gcw.empty() {
				restart = true
				break
			}
		}
	})
	if restart {
		getg().m.preemptoff = ""
		systemstack(func() {
			work.cpuStats.accumulateGCPauseTime(nanotime()-stw.finishedStopping, work.maxprocs)

			now := startTheWorldWithSema(0, stw)
			work.pauseNS += now - stw.startedStopping
		})
		semrelease(&worldsema)
		goto top
	}

	// 终止标记
	gcMarkTermination(stw)
}
```

### gcMarkTermination

```go
func gcMarkTermination(stw worldStop) {
	// 设置 gc 终止阶段
	setGCPhase(_GCmarktermination)

	// 修改 gc goroutine 状态
	casGToWaitingForGC(curgp, _Grunning, waitReasonGarbageCollection)

	// 在 g0 栈上运行 gc。这样做是为了保证我们当前运行的 g 栈不再变化，
	// 这有助于减少根集大小（g0 栈不会被扫描，我们也不需要扫描 gc 的内部状态）。
	// 我们还需要切换到 g0 以便可能的栈缩减。 
	systemstack(func() {
		gcMark(startTime)
	})

	var stwSwept bool
	systemstack(func() {
		work.heap2 = work.bytesMarked
		if debug.gccheckmark > 0 {
			// Run a full non-parallel, stop-the-world
			// mark using checkmark bits, to check that we
			// didn't forget to mark anything during the
			// concurrent mark process.
			startCheckmarks()
			gcResetMarkState()
			gcw := &getg().m.p.ptr().gcw
			gcDrain(gcw, 0)
			wbBufFlush1(getg().m.p.ptr())
			gcw.dispose()
			endCheckmarks()
		}

		// 标记完成，我们可以关闭写屏障。
		setGCPhase(_GCoff)
        // 开启清扫
		stwSwept = gcSweep(work.mode)
	})

	casgstatus(curgp, _Gwaiting, _Grunning)

	// ......

	systemstack(func() {
        // Start The World
		startTheWorldWithSema(now, stw)
	})


	// 确保所有 mcaches 被清空。每个 P 在分配前将清空自己的 mcache，但空闲的 P 可能不会。
	// 由于这对于扫描所有 spans 是必需的，我们需要确保在开始下一个 GC 周期前，所有 mcaches 都已被清空。
	forEachP(waitReasonFlushProcCaches, func(pp *p) {
		pp.mcache.prepareForSweep()
		if pp.status == _Pidle {
			systemstack(func() {
				lock(&mheap_.lock)
				pp.pcache.flush(&mheap_.pages)
				unlock(&mheap_.lock)
			})
		}
		pp.pinnerCache = nil
	})
	if sl.valid {
		// Now that we've swept stale spans in mcaches, they don't
		// count against unswept spans.
		//
		// Note: this sweepLocker may not be valid if sweeping had
		// already completed during the STW. See the corresponding
		// begin() call that produced sl.
		sweep.active.end(sl)
	}

}
```

## 清扫 

```go
func gcSweep(mode gcMode) bool {
	// ......
	
	// 并发清扫
	lock(&sweep.lock)
	if sweep.parked {
		sweep.parked = false
		ready(sweep.g, 0, true)
	}
	unlock(&sweep.lock)
	return false
}

```

sweep.g 的创建 

```go
var sweep sweepdata

type sweepdata struct {
	g      *g
}

// 在 runtime main 中调用了gcenable()
func gcenable() {
	c := make(chan int, 2)
	go bgsweep(c)
	go bgscavenge(c)
	<-c
	<-c
	memstats.enablegc = true 
}
```

### bgscavenge

从 OS 申请到 _heap 上的内存 不用了 还回去一些

```go
func bgscavenge(c chan int) {
	scavenger.init()

	c <- 1
	// 等待
	scavenger.park()

	for {
		// 执行一次清理 (scavenge) 操作，返回释放的内存量和本次操作的耗时
		released, workTime := scavenger.run()
		// 没有释放内存，休眠
		if released == 0 {
			scavenger.park()
			continue
		}
		// 如果释放了内存，将释放的内存量和耗时记录到统计信息中
		mheap_.pages.scav.releasedBg.Add(released)
		scavenger.sleep(workTime)
	}
}
```
####  scavenger.run

```go
func (s *scavengerState) run() (released uintptr, worked float64) {
	for worked < minScavWorkTime {
		// 如果需要停止，就停止
		if s.shouldStop() {
			break
		}

		const scavengeQuantum = 64 << 10
		r, duration := s.scavenge(scavengeQuantum)

		const approxWorkedNSPerPhysicalPage = 10e3
		if duration == 0 {
			worked += approxWorkedNSPerPhysicalPage * float64(r/physPageSize)
		} else {
			worked += float64(duration)
		}
		released += r

	}
	
	return
}
```

#### s.scavenge

```go
if s.scavenge == nil {
		s.scavenge = func(n uintptr) (uintptr, int64) {
			start := nanotime()
			r := mheap_.pages.scavenge(n, nil, false)
			end := nanotime()
			if start >= end {
				return r, 0
			}
			scavenge.backgroundTime.Add(end - start)
			return r, end - start
		}
	}
```

```go
func (p *pageAlloc) scavenge(nbytes uintptr, shouldStop func() bool, force bool) uintptr {
	released := uintptr(0)
	for released < nbytes {
		ci, pageIdx := p.scav.index.find(force)
		if ci == 0 {
			break
		}
		systemstack(func() {
			released += p.scavengeOne(ci, pageIdx, nbytes-released)
		})
		if shouldStop != nil && shouldStop() {
			break
		}
	}
	return released
}
```

#### scavengeOne

```go
func (p *pageAlloc) scavengeOne(ci chunkIdx, searchIdx uint, max uintptr) uintptr {
	maxPages := max / pageSize
	if max%pageSize != 0 {
		maxPages++
	}

	minPages := physPageSize / pageSize
	if minPages < 1 {
		minPages = 1
	}

	lock(p.mheapLock)
	if p.summary[len(p.summary)-1][ci].max() >= uint(minPages) {
		// 找到回收的开始地址 和页数
		base, npages := p.chunkOf(ci).findScavengeCandidate(searchIdx, minPages, maxPages)

		// If we found something, scavenge it and return!
		if npages != 0 {
			// ......
			unlock(p.mheapLock)

			if !p.test {
				// 系统调用
				sysUnused(unsafe.Pointer(addr), uintptr(npages)*pageSize)

				// ......
			}

			// 更新统计信息
			lock(p.mheapLock)
			if b := (offAddr{addr}); b.lessThan(p.searchAddr) {
				p.searchAddr = b
			}
			p.chunkOf(ci).free(base, npages)
			p.update(addr, uintptr(npages), true, false)

			// Mark the range as scavenged.
			p.chunkOf(ci).scavenged.setRange(base, npages)
			unlock(p.mheapLock)

			return uintptr(npages) * pageSize
		}
	}
	p.scav.index.setEmpty(ci)
	unlock(p.mheapLock)

	return 0
}
```

### bgsweep

```go
func bgsweep(c chan int) {
	sweep.g = getg()

	lockInit(&sweep.lock, lockRankSweep)
	lock(&sweep.lock)
	sweep.parked = true
	c <- 1
	// 休眠 等待gc 完成 唤醒它
	goparkunlock(&sweep.lock, waitReasonGCSweepWait, traceBlockGCSweep, 1)

	for {
		// 执行一次清扫操作
		for sweepone() != ^uintptr(0) {
			nSwept++
			// 每清扫一定数量对象，就尝试让出 CPU
			if nSwept%sweepBatchSize == 0 {
				goschedIfBusy()
			}
		}
		// 释放一些写缓冲区
		for freeSomeWbufs(true) {
			goschedIfBusy()
		}
		lock(&sweep.lock)
		// 检查是否完成了所有扫描任务。
		if !isSweepDone() {
			unlock(&sweep.lock)
			continue
		}
		sweep.parked = true
		// 休眠 等待下一次 gc 标记完成唤醒它
		goparkunlock(&sweep.lock, waitReasonGCSweepWait, traceBlockGCSweep, 1)
	}
}
```

#### sweepone

```go
func sweepone() uintptr {
	gp := getg()
	gp.m.locks++
	sl := sweep.active.begin()
	if !sl.valid {
		gp.m.locks--
		return ^uintptr(0)
	}

	// Find a span to sweep.
	npages := ^uintptr(0)
	var noMoreWork bool
	for {
		// 查找 span
		s := mheap_.nextSpanForSweep()
		if s == nil {
			noMoreWork = sweep.active.markDrained()
			break
		}
		// 试着清扫
		if s, ok := sl.tryAcquire(s); ok {
			// Sweep the span we found.
			npages = s.npages
			if s.sweep(false) {
				mheap_.reclaimCredit.Add(npages)
			} else {
				npages = 0
			}
			break
		}
	}
	sweep.active.end(sl)

	// ......
	gp.m.locks--
	return npages
}
```

sweep 主要逻辑是把 刚刚标记过的 gcmarkBits 复制给 allocBits 

```go
func (sl *sweepLocked) sweep(preserve bool) bool {
	// ......
	s.allocBits = s.gcmarkBits
	s.gcmarkBits = newMarkBits(uintptr(s.nelems))
	// ......
}
```

#### 分配内存

如果我们的 span 标记之后 还没来得及清扫，修改了 allocBits 值，然后再清扫会出问题。go 是怎么解决的呢？

```go
// 拿 span 的时候会检查 如果没清扫就先清扫一下 再使用内存
func (c *mcentral) cacheSpan() *mspan {
	// ......
	if s, ok := sl.tryAcquire(s); ok {
		// We got ownership of the span, so let's sweep it.
		s.sweep(true)
	}
	// ......
}

func (h *mheap) alloc(npages uintptr, spanclass spanClass) *mspan {
	var s *mspan
	systemstack(func() {
		if !isSweepDone() {
			h.reclaim(npages)
		}
		s = h.allocSpan(npages, spanAllocHeap, spanclass)
	})
	return s
}

func (h *mheap) reclaim(npage uintptr) {
	// ......
	nfound := h.reclaimChunk(arenas, idx, pagesPerReclaimerChunk)
	// ......
}

func (h *mheap) reclaimChunk(arenas []*pageAlloc, idx, npages uintptr) uintptr {
	// ......
	if s.sweep(false) {
	}
	// ......
}

```