# Golang 面试驱动学习


![header_](http://image.p1nant0m.com/header_opensource.png)

## 说在前面
🐹 **WIP** 

### Go 运行时行为
🐹 **WIP** 

#### GC 触发时机
GC 触发的入口为 `runtime.gcStart()` 垃圾收集被触发后会通过 `runtime.gcTrigger.test()` 方法检测当前是否需要触发垃圾收集，该方法会根据三种方式触发不同的检查，

- gcTriggerHeap：堆内存分配达到控制器计算的触发堆大小
- gcTriggerTime：如果一段时间内没有触发，就会触发新一轮的循环，该触发条件由 runtime.forcegcperiod 变量控制，默认时间为 2 分钟
- gcTriggerperiod：如果当前没有开启垃圾收集，则触发新的循环

所有出现 runtime.gcTrigger 结构体的位置都是触发垃圾收集的代码位置

- runtime.sysmon 和 runtime.forcegchelper —— 后台运行定时检查和垃圾收集；（gcTriggerTime）
- runtime.GC —— 用户程序手动触发垃圾收集；（gcTriggerCycle）
- runtime.mallocgc —— 申请内存时根据堆大小触发垃圾收集 （ gcTriggerHeap）

{{< admonition type=tip title="🤔 你该怎么回答？" open=false >}}

GC 触发时机按照触发的行为主体可以大致分为两类，一类是由用户主动触发，另一类是运行时系统依据某些条件自动触发。

- 用户触发：用户可以在用户代码中通过调用 `runtime.GC()` ，以 `gcTriggerCycle` 充当谓词构建 `gcTrigger` 结构体，调用 `runtime.gcStart()` 触发垃圾收集；（这里会不会真正触发垃圾收集还要看当前是否就是处在垃圾收集的阶段，也就是说用户手动触发垃圾收集前，GC 已经启动了。这将通过 `runtime.gcTrigger.test()` 方法进行检测）

- 系统监控触发：在 `runtime.sysmon` 与 `runtime.forcegchelper` 中会以 `gcTriggerTime` 充当谓词构建 `gcTrigger` 结构体，调用 `(gcTrigger).test()`  方法检测当前是否需要触发一次 GC。若距离上一次 GC 的时间超过了一定的阈值（由 `forcegcperiod` 变量决定，该变量的默认值是 2 * 60 * 1e9 也就是两分钟），则会触发一次 GC。

- 由 `mallocgc` 触发：在 `runtime.mallocgc()` 代码的结尾会以 `gcTriggerHeap` 充当谓词构建 `gcTrigger` 结构体，调用 `(gcTrigger).test()` 方法检测当前是否需要触发一次GC。若当前堆的大小大于通过**步调算法**计算得出的下一次 GC 触发大小，则会触发一次 GC，在代码中为 `gcController.heapLive >= gcController.trigger` 时 `(gcTrigger).test()` 方法返回 **True**。步调算法核心思想是控制内存增长的比例，可以通过 `GOGC` 和 `debug.SetGCPercent` 进行控制。其通过计算得出的下一次 GC 时堆大小的阈值 `gcController.trigger`，来控制 GC 触发的时间点，进而保证最后 GC 结束时堆大小应该小于算法得出的目标堆大小。下图展示的是这个过程，

{{< image src="http://image.p1nant0m.com/202209261110433.png" caption="步调 (Pacing) 算法实现目标" src_s="http://image.p1nant0m.com/202209261110433.png" src_l="http://image.p1nant0m.com/202209261110433.png" >}}

{{< /admonition >}}

#### g 的调度流程

我们使用 `go` 关键字可以轻松创建一个 goroutine, 并在其中执行我们的用户代码，快速地实现并发编程。这简单的用户层面使用语法，背后是运行时调度系统做的一系列复杂的工作，这里简单的描述一下相关的过程。

Go 语言中的调度流程与 GMP 模型的设计息息相关，但本文这里并不会去介绍相关 GMP 模型的内容，我们更希望将关注的焦点放在运行时的调度行为上。用户使用 `go` 关键字创建一个 goroutine 后，运行时会对其进行一系列的初始化操作，并将其加入到相关 P 的本地可运行队列中，等待一下次调度循环中被选中执行。我们关键就来看一看这一个 `runtime.schedule` 函数具体都干了些什么工作。

`runtime.schedule` 将主要的寻找可运行 g 的逻辑写到了 `runtime.findRunnable` 函数中，其会从下面几处查找待执行的 goroutine，

- 为了保证公平，当全局运行队列中有待执行的 Goroutine 时，通过 schedtick 保证有一定概率（从源代码中看，每过61次调度循环便会主动先去 GRQ 中找是否有可运行的 g）会从全局的运行队列中查找对应的 Goroutine；
```go
	// Check the global runnable queue once in a while to ensure fairness.
	// Otherwise two goroutines can completely occupy the local runqueue
	// by constantly respawning each other.
	if _p_.schedtick%61 == 0 && sched.runqsize > 0 {
		lock(&sched.lock)
		gp = globrunqget(_p_, 1)
		unlock(&sched.lock)
		if gp != nil {
			return gp, false, false
		}
	}
```

- 从处理器（P）的 LRQ 中查找待执行的 Goroutine；
```go
	// local runq
	if gp, inheritTime := runqget(_p_); gp != nil {
		return gp, inheritTime, false
	}
```

- 从 GRQ 中查找待执行的 Goroutine；
```go
	// global runq
	if sched.runqsize != 0 {
		lock(&sched.lock)
		gp := globrunqget(_p_, 0)
		unlock(&sched.lock)
		if gp != nil {
			return gp, false, false
		}
	}
```

- 从网络轮询器中查找是否有 Goroutine 等待运行；
```go
	// Poll network.
	// This netpoll is only an optimization before we resort to stealing.
	// We can safely skip it if there are no waiters or a thread is blocked
	// in netpoll already. If there is any kind of logical race with that
	// blocked thread (e.g. it has already returned from netpoll, but does
	// not set lastpoll yet), this thread will do blocking netpoll below
	// anyway.
	if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
		if list := netpoll(0); !list.empty() { // non-blocking
			gp := list.pop()
			injectglist(&list)
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.enabled {
				traceGoUnpark(gp, 0)
			}
			return gp, false, false
		}
	}
```

- 通过 runtime.runqsteal 尝试从其他随机的处理器（P）中窃取待运行的 Goroutine，该函数还可能窃取处理器（P）的计时器。
```go
	// Spinning Ms: steal work from other Ps.
	//
	// Limit the number of spinning Ms to half the number of busy Ps.
	// This is necessary to prevent excessive CPU consumption when
	// GOMAXPROCS>>1 but the program parallelism is low.
	procs := uint32(gomaxprocs)
	if _g_.m.spinning || 2*atomic.Load(&sched.nmspinning) < procs-atomic.Load(&sched.npidle) {
		if !_g_.m.spinning {
			_g_.m.spinning = true
			atomic.Xadd(&sched.nmspinning, 1)
		}

		gp, inheritTime, tnow, w, newWork := stealWork(now)
		now = tnow
		if gp != nil {
			// Successfully stole.
			return gp, inheritTime, false
		}
		if newWork {
			// There may be new timer or GC work; restart to
			// discover.
			goto top
		}
		if w != 0 && (pollUntil == 0 || w < pollUntil) {
			// Earlier timer to wait for.
			pollUntil = w
		}
	}
```

如果上述过程都没有找到一个能够执行的 goroutine 的话，那么就意味着目前的处理器（P）十分有空并且当前整个程序运行的工作负载压力也不大，这时 `runtime.schedule` 就会去尝试干其它工作。如果我们这时正处于 **GC mark** 阶段的话，处理器就可以运行一个任务帮助进行对象的扫描与标记。
```go
	// If we're in the GC mark phase, can safely scan and blacken objects,
	// and have work to do, run idle-time marking rather than give up the P.
	if gcBlackenEnabled != 0 && gcMarkWorkAvailable(_p_) && gcController.addIdleMarkWorker() {
		node := (*gcBgMarkWorkerNode)(gcBgMarkWorkerPool.pop())
		if node != nil {
			_p_.gcMarkWorkerMode = gcMarkWorkerIdleMode
			gp := node.gp.ptr()
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.enabled {
				traceGoUnpark(gp, 0)
			}
			return gp, false, false
		}
		gcController.removeIdleMarkWorker()
	}
```

如果当前确实没有任何工作可以进行，P 就可以“休息”了。调度器会将其放入空闲队列中，等待有任务需要处理时再将其调度出来，在这之前先要对操作上锁，并查看全局队列中是否出现了可以执行的任务，若有可以执行的任务处理器（P）便不会放入空闲队列中，转而去执行这个获取到的任务，否则，处理器（P）将被放入空闲队列。
```go
	// return P and block
	lock(&sched.lock)
	if sched.gcwaiting != 0 || _p_.runSafePointFn != 0 {
		unlock(&sched.lock)
		goto top
	}
	if sched.runqsize != 0 {
		gp := globrunqget(_p_, 0)
		unlock(&sched.lock)
		return gp, false, false
	}
	if releasep() != _p_ {
		throw("findrunnable: wrong p")
	}
	pidleput(_p_)
	unlock(&sched.lock)
```

在上述这些工作完成后就应该快进入到休眠阶段了，不过这时候调度器仍然还在尝试寻找一个可运行的任务，通过 `checkRunqsNoP()` 函数查找所有处理器（P）的 LRQ 中是否有待执行的 goroutines，以及 `checkIdleGCNoP` 函数查找进行 GC 任务的 Worker 是否还未被运行。
```go
		// Check all runqueues once again.
		_p_ = checkRunqsNoP(allpSnapshot, idlepMaskSnapshot)
		if _p_ != nil {
			acquirep(_p_)
			_g_.m.spinning = true
			atomic.Xadd(&sched.nmspinning, 1)
			goto top
		}

		// Check for idle-priority GC work again.
		_p_, gp = checkIdleGCNoP()
		if _p_ != nil {
			acquirep(_p_)
			_g_.m.spinning = true
			atomic.Xadd(&sched.nmspinning, 1)

			// Run the idle worker.
			_p_.gcMarkWorkerMode = gcMarkWorkerIdleMode
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.enabled {
				traceGoUnpark(gp, 0)
			}
			return gp, false, false
		}
```

最后的最后，调度器还会尝试从网络轮询器中查找是否已经有准备好的 goroutine 等待执行，如果有的话就从空闲 P 队列中取出一个处理器，并返回队列中一个待执行的 goroutine。如果实在确实是找不到工作的话，就执行 `stopm` 将当前线程挂起进入睡眠。
```go
		list := netpoll(delay) // block until new work is available
		atomic.Store64(&sched.pollUntil, 0)
		atomic.Store64(&sched.lastpoll, uint64(nanotime()))
		if faketime != 0 && list.empty() {
			// Using fake time and nothing is ready; stop M.
			// When all M's stop, checkdead will call timejump.
			stopm()
			goto top
		}
		lock(&sched.lock)
		_p_ = pidleget()
		unlock(&sched.lock)
		if _p_ == nil {
			injectglist(&list)
		} else {
			acquirep(_p_)
			if !list.empty() {
				gp := list.pop()
				injectglist(&list)
				casgstatus(gp, _Gwaiting, _Grunnable)
				if trace.enabled {
					traceGoUnpark(gp, 0)
				}
				return gp, false, false
			}
```

重新回到 `runtime.schedule` 函数中，当 `runtime.findRunnable` 找到一个可以执行的 Goroutine 后，便会由 `runtime.execute` 执行获取到的 Goroutine，其会对 Goroutine 的 g 结构体做一些初始化工作，最后通过 `runtime.gogo` 将 Goroutine 调度到当前线程上，
```go
// Schedules gp to run on the current M.
// If inheritTime is true, gp inherits the remaining time in the
// current time slice. Otherwise, it starts a new time slice.
// Never returns.
//
// Write barriers are allowed because this is called immediately after
// acquiring a P in several places.
//
//go:yeswritebarrierrec
func execute(gp *g, inheritTime bool) {
	_g_ := getg()

	// Assign gp.m before entering _Grunning so running Gs have an
	// M.
	_g_.m.curg = gp
	gp.m = _g_.m
	casgstatus(gp, _Grunnable, _Grunning)
	gp.waitsince = 0
	gp.preempt = false
	gp.stackguard0 = gp.stack.lo + _StackGuard
	if !inheritTime {
		_g_.m.p.ptr().schedtick++
	}

	// Check whether the profiler needs to be turned on or off.
	hz := sched.profilehz
	if _g_.m.profilehz != hz {
		setThreadCPUProfiler(hz)
	}

	if trace.enabled {
		// GoSysExit has to happen when we have a P, but before GoStart.
		// So we emit it here.
		if gp.syscallsp != 0 && gp.sysblocktraced {
			traceGoSysExit(gp.sysexitticks)
		}
		traceGoStart()
	}

	gogo(&gp.sched)
}
```

`runtime.gogo` 是一段用 Go 汇编写成的函数，在不同的处理器架构上有不同的实现，不过总体的实现大同小异，下面是该函数在 **amd64** 架构上的实现，
```go
// func gogo(buf *gobuf)
// restore state from Gobuf; longjmp
TEXT runtime·gogo(SB), NOSPLIT, $0-8
	MOVQ	buf+0(FP), BX		// gobuf
	MOVQ	gobuf_g(BX), DX
	MOVQ	0(DX), CX		// make sure g != nil
	JMP	gogo<>(SB)

TEXT gogo<>(SB), NOSPLIT, $0
	get_tls(CX)
	MOVQ	DX, g(CX)
	MOVQ	DX, R14		// set the g register
	MOVQ	gobuf_sp(BX), SP	// restore SP
	MOVQ	gobuf_ret(BX), AX
	MOVQ	gobuf_ctxt(BX), DX
	MOVQ	gobuf_bp(BX), BP
	MOVQ	$0, gobuf_sp(BX)	// clear to help garbage collector
	MOVQ	$0, gobuf_ret(BX)
	MOVQ	$0, gobuf_ctxt(BX)
	MOVQ	$0, gobuf_bp(BX)
	MOVQ	gobuf_pc(BX), BX // 获取待执行任务函数的程序计数器
	JMP	BX // 执行传入 goroutine 的任务代码
```

其中 `runtime.gobuf.sp` 中存放了 `runtime.goexit` 的程序计数器，并被恢复到了当前栈 SP 位置上，当目标函数返回后，会从堆中查找调用的地址并跳转回调用方继续执行剩余代码（`runtime.goexit`）。
```go
// The top-most function running on a goroutine
// returns to goexit+PCQuantum.
TEXT runtime·goexit(SB),NOSPLIT|TOPFRAME,$0-0
	BYTE	$0x90	// NOP
	CALL	runtime·goexit1(SB)	// does not return
	// traceback from goexit1 must hit code range of goexit
	BYTE	$0x90	// NOP
```

其中 `runtime.goexit1` 的实现如下，
```go
// Finishes execution of the current goroutine.
func goexit1() {
	if raceenabled {
		racegoend()
	}
	if trace.enabled {
		traceGoEnd()
	}
	mcall(goexit0)
}
```

通过 `mcall` 其会在当前线程的 g0 的栈上调用 `runtime.goexit0` 函数，该函数会设置 Goroutine 相关状态、清理其中的一些字段、移除 Goroutine 和线程的关联，并调用 `runtime.gfput` 重新加入处理器的 Goroutine 空闲列表 gFree，最终重新调用 `runtime.schedule` 函数触发新一轮的 Goroutine 调度循环！
```go
func goexit0(gp *g) {
	_g_ := getg()
	_p_ := _g_.m.p.ptr()

	casgstatus(gp, _Grunning, _Gdead) // Goroutine 状态转移
	gcController.addScannableStack(_p_, -int64(gp.stack.hi-gp.stack.lo))
	if isSystemGoroutine(gp, false) {
		atomic.Xadd(&sched.ngsys, -1)
	}
	gp.m = nil
	locked := gp.lockedm != 0
	gp.lockedm = 0
	_g_.m.lockedg = 0
	gp.preemptStop = false
	gp.paniconfault = false
	gp._defer = nil // should be true already but just in case.
	gp._panic = nil // non-nil for Goexit during panic. points at stack-allocated data.
	gp.writebuf = nil
	gp.waitreason = 0
	gp.param = nil
	gp.labels = nil
	gp.timer = nil

	if gcBlackenEnabled != 0 && gp.gcAssistBytes > 0 {
		// Flush assist credit to the global pool. This gives
		// better information to pacing if the application is
		// rapidly creating an exiting goroutines.
		assistWorkPerByte := gcController.assistWorkPerByte.Load()
		scanCredit := int64(assistWorkPerByte * float64(gp.gcAssistBytes))
		atomic.Xaddint64(&gcController.bgScanCredit, scanCredit)
		gp.gcAssistBytes = 0
	}

	dropg() // 移除当前 Goroutine 与线程的关联

	if GOARCH == "wasm" { // no threads yet on wasm
		gfput(_p_, gp)
		schedule() // never returns
	}

	if _g_.m.lockedInt != 0 {
		print("invalid m->lockedInt = ", _g_.m.lockedInt, "\n")
		throw("internal lockOSThread error")
	}
	gfput(_p_, gp) // 将 Goroutine 键入处理器的 Goroutine 空闲列表 gFree
	if locked {
		// The goroutine may have locked this thread because
		// it put it in an unusual kernel state. Kill it
		// rather than returning it to the thread pool.

		// Return to mstart, which will release the P and exit
		// the thread.
		if GOOS != "plan9" { // See golang.org/issue/22227.
			gogo(&_g_.m.g0.sched)
		} else {
			// Clear lockedExt on plan9 since we may end up re-using
			// this thread.
			_g_.m.lockedExt = 0
		}
	}
	schedule() // 重新触发新一轮的调度循环
}
```

至此，我们可以说 Goroutine 的调度没有结束，其永远不会返回，只会触发一轮又一轮的调度循环。


## References
[The Go Blog](https://go.dev/blog/)

[The Golang Repo](https://github.com/golang/go)

[Go 程序员面试笔试宝典](https://golang.design/go-questions/)

[Go 语言原本](https://golang.design/under-the-hood/)

[Go 语言设计与实现](https://draveness.me/golang/)

[The Go Programming Language](https://beyondkmp.com/books/golang/The.Go.Programming.Language.pdf)

