# 大白话讲 Golang Channel 实现


![header_](https://image.p1nant0m.xyz/header_opensource.png)

### 大白话讲 Golang Channel 实现
本文尝试以轻松的口吻介绍 Golang 中 Channel 的相关实现，我们会以源码不断迭代发展的视角去观察 Channel 实现的变化，具体的来说，通过 Go 主仓库 Commit 历史记录来查看每一次改动所对应的 Issues，一同来看下 Go 开发团队的具体考量。除此之外，我们也需要抓住一些不变的本质，这些本质的实现是整个 Channel 实现设计的基础，考察相关的数据结构以及方法对我们理解具体编码时 Channel 的行为颇有裨益。

本文是笔者在学习过程中写成的，参考了许多前人写就的文章以及见解，为了让本文与其它介绍 Go Channel 相关实现的技术文章形成差异，我将以一种轻松非完全技术的口吻来介绍相关的实现，尽量减轻自身与读者进行二次理解时的认知负担。

#### 本文摘要
本文将从介绍 Golang 的并发哲学开始，讲述 Go 对于并发编程具有“魅力”的设计实现，这为程序员提供了一种十分轻量及低认知负担地进行并发编程的方式，使得程序员能从关心如线程调度、线程使用开销、内存同步访问控制等底层问题中解放出来，来将更多的时间花费到解决更具体地业务问题上。

接着我们将会从四个个方面去考察 Channel 的设计实现，
- 创建一个 Channel
- 向 Channel 中发送数据
- 从 Channel 中取出数据
- 关闭 Channel

{{< admonition type=note title="胡思乱想" open=false >}}
创建一个 Channel 解决的是一个对象有无的问题，向 Channel 中传入数据与取出数据是 Channel 的功能性实现，而关闭 Channel 则是有始有终，是对有限资源的回收。
{{< /admonition >}}

最后，我们会沿着历史的视角，观察 Channel 实现的版本迭代以及其背后的考量。

值得一提的是，源码中 Channel 主要逻辑实现的位置：**runtime package** [chan.go](https://github.com/golang/go/blob/master/src/runtime/chan.go)





#### Golang 并发哲学

Go 语言中最令人津津乐道的特性无外乎是对并发的天然支持了。使用过 Go 进行并发编程的同学们一定能够体会到使用 `go` 关键字创建协程并通过 `chan` 进行跨协程通信的魅力。goroutine 轻量化的特性让我们能够无心理负担地使用 `go` 关键字轻松创建上万个协程的同时仍能保持一个较高的性能，使用 `chan` 进行跨协程间通信有助于轻松实现生产者-消费者模型，充分利用现代机器并行计算的能力。同时，尽量避免通过使用共享内存的方式来进行跨协程之间的通信。Golang 对此还有一句[广为人知的格言](https://go.dev/blog/codelab-share)，

> Do not communicate by sharing memory; instead, share memory by communicating.

而这一切都是 Golang 语言对 CSP (Communicating Sequential Process) 模型的诠释。1978 年 C.A.R Hoare [发表在 ACM 的一篇论文](https://www.cs.cmu.edu/~crary/819-f09/Hoare78.pdf)中指出一门编程语言应该重视 input 和 output，尤其是注重并发编程。Go 吸收了论文中有益的设计以及观点，并将其中的并发编程哲学内化到整个语言的设计当中，进一步将 CSP 发扬光大。Go 也并没有抛弃传统的内存同步访问控制，因其仍然存在许多的使用场景，在 Go 中也有相应的 sync 包提供支持。

Goroutine 与 Channel 相互搭配解决了并发编程中棘手的问题，程序员无需再关心与线程调度、共享内存同步访问控制、线程使用开销等底层问题，他们仅需要关心具体的业务逻辑，充分发挥自己的想象力利用 Go 的能力去解决现实世界中的问题。

{{< admonition type=tip title="浅谈 goroutine 与 channel" open=false >}}
goroutine 可以理解为一种粒度更细的线程，它由 Go 运行时维护，负责它的创建、调度、销毁全生命周期过程。它扮演了并发编程中工作者的角色，维护了在其管理域下代码执行的上下文环境，提供了轻量化的资源集合，拥有更小的上下文切换开销，是 Go 中实现高性能并发编程的基础对象。

Channel 则解决了 goroutine 之间数据传递的问题。利用 Channel 我们可以实现以下但不限于这些应用：（1）停止信号；（2）定时任务；（3）解耦生产方和消费方；（4）控制并发数；

完成一项具体的工作需要不同部门的通力配合，每一个阶段都有相应的投入与产出，某一阶段可能是是单一重复劳动，增加工人的数量能够显著提高效率，使用流水线的作业方式能够执行同一生产过程的不同阶段，也能够提高最终的生产效率。goroutine 好比完成任务的工人，Channel 则是贯通不同部门的货物传输管道，两者相互配合最终实现 CPU 时间片的高效利用。
{{< /admonition >}}

#### 1️⃣ 创建一个 Channel

在 Go 中，我们通常使用以下代码创建一个 Channel，

```go
myChan := make(chan Foo) // Channel without buffer
myChanWithBuf := make(chan Foo, 10) // Channel with buffer
```

上述两行代码分别创建了两种不同类型的 Channel，一种是不带有缓冲区的 Channel，一种是带有缓冲区的 Channel，它们分别对应于上述代码段的 `1-2` 行。

这是使用 Go 语言在语用层面对 Channel 进行定义，在 Go 编译器进行编译的类型检查阶段中，会根据代表 `make` 关键字的 **OMAKE**  节点参数类型的不同来转换成三种不同类型的节点（回想我们使用 `make` 关键字创建切片、哈希表以及这里的 Channel），分别是：**OMAKESLICE**，**OMAKEMAP**，**OMAKECHAN**，最后调用不同的运行时函数来创建相应节点的数据结构。

我们这里主要介绍创建 Channel 时，运行时主要的执行逻辑，`makechan`。不过，在介绍 `makechan` 的具体实现之前，我们得先来看一看运行时中 `chan` 所对应的底层数据结构，`hchan`，它的结构如下代码段所示，

```go
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}
```

这里我们逐一介绍每一个字段的具体作用，
- `qcount` 指的是当前循环队列中的元素个数
- `dataqsiz` 指的是循环队列的最大长度，也就是在创建 `chan` 时通过 `make` 关键字传入的第二个参数
- `buf` 是一个指向底层实现循环队列数组的指针
- `elemsize` 指的是创建 `chan` 时给定 `chan` 中数据类型的大小（比如 `int` 为 4 个字节，在这里便是 4）
- `closed` 是一个标志位，用于指示当前 `chan` 是否处于关闭状态，当 `chan` 关闭时，其值为 1
- `elemtype` 指的是 `chan` 中的数据类型
- `sendx` 指的是循环队列中待发送元素的下标
- `recvx` 指的是循环队列中待接收元素应该存放位置的下标
- `recvq` 是一个双向循环链表，其中存放的是等待接收数据的 goroutine。
- `sendq` 是一个双向循环链表，其中存放的是等待发送数据的 goroutine。
- `lock` 是一个互斥锁，用于保护上述字段以及 `sudog`  结构体中的字段在进入临界区时的读写。

我们在这里再多花一点时间来看一看 `recvq` 和 `sendq`，它们对应的数据类型是 `waitq` 如下代码片段所示，
```go
type waitq struct {
	first *sudog
	last  *sudog
}
```
`waitq` 结构体中有两个指向 `sudog` 结构体的指针，其中 `first` 指向链表中的首个元素，`last` 指向链表中的最后一个元素。当链表为空时，`first` 与 `last` 都为 `nil`。其中 `sudog` 是这样的一个结构体，

```go
// sudog represents a g in a wait list, such as for sending/receiving
// on a channel.
//
// sudog is necessary because the g ↔ synchronization object relation
// is many-to-many. A g can be on many wait lists, so there may be
// many sudogs for one g; and many gs may be waiting on the same
// synchronization object, so there may be many sudogs for one object.
//
// sudogs are allocated from a special pool. Use acquireSudog and
// releaseSudog to allocate and free them.
type sudog struct {
	// The following fields are protected by the hchan.lock of the
	// channel this sudog is blocking on. shrinkstack depends on
	// this for sudogs involved in channel ops.

	g *g

	next *sudog
	prev *sudog
	elem unsafe.Pointer // data element (may point to stack)

	// The following fields are never accessed concurrently.
	// For channels, waitlink is only accessed by g.
	// For semaphores, all fields (including the ones above)
	// are only accessed when holding a semaRoot lock.

	acquiretime int64
	releasetime int64
	ticket      uint32

	// isSelect indicates g is participating in a select, so
	// g.selectDone must be CAS'd to win the wake-up race.
	isSelect bool

	// success indicates whether communication over channel c
	// succeeded. It is true if the goroutine was awoken because a
	// value was delivered over channel c, and false if awoken
	// because c was closed.
	success bool

	parent   *sudog // semaRoot binary tree
	waitlink *sudog // g.waiting list or semaRoot
	waittail *sudog // semaRoot
	c        *hchan // channel
}
```

`sudog` 是 `g` 的一个封装，将因等待 channel 相关发送/接收事件而陷入睡眠的 `goroutine` 串联成一个双向链表。

接下来，我们便正式介绍 `makechan`，其函数签名如下所示，

```go
func makechan(*chantype, int) *hchan

```

其中创建 `hchan` 的主要逻辑如下所示，
```go
	var c *hchan
	switch {
	case mem == 0:
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// Elements contain pointers.
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	lockInit(&c.lock, lockRankHchan)
```

其中根据创建 `chan` 时具体的入参，有三种分配 `hchan` 相关内存的情况，
- 创建无缓冲区的 channel 或者创建的 `chan` 中传递的类型为 `struct{}`，也即一个空的结构体类型（其占用的内存空间为 0）。
- 创建的 `chan` 中传递的类型元素不含指针（其本身不是指针，或者结构体中不含有指针类型）
- 创建的 `chan` 中传递的类型元素包含有指针（其本身是指针，或者结构体中含有指针类型）

在第一种情况下，我们直接分配大小为 hchanSize 的内存空间，`c.buf` 由于没有被使用到所以我们并不需要为其进行内存分配。

在第二种情况下，我们需要分配大小为 `hchanSize + mem` 的内存空间，其中 `mem` 是在 `mem, overflow := math.MulUintptr(elem.size, uintptr(size))` 中计算得到的，其代表的是 `chan` 中传输的元素类型大小乘上 `buf` 的大小，也即为 `buf` 分配一段线性连续的内存空间，并且该内存位置与 `hchan` 结构体相邻，最后我们将 `c.buf` 指向这段连续内存空间的首部（其实也就是一个数组）。

在第三种情况下，分别分配 `hchan` 结构体以及 `c.buf` 的内存空间，由于 `chan` 中的元素类型中包含有指针类型，GC 需要扫描这段内存空间以确认是否需要进行垃圾回收工作。在 `runtime.mallocgc` 中我们可以找到相应的代码段来支撑我们的结论，

```go
noscan := typ == nil || typ.ptrdata == 0
```

{{< admonition type=note title="为什么要进行两段内存分配？" open=false >}}
由于 `chan` 中传递的元素类型中包含有指针，GC 需要对这段内存空间进行扫描以确认是否有可回收的内存空间，而 `hchan` 属于 runtime 中的结构，其生命周期由 runtime 内编码逻辑决定，因此 GC 不参与其有关的内存回收工作。GC 需要对用户代码创建的内存对象进行检查，以防出现内存泄露的情况，这也是 GC 的职责。

两次进行内存分配的工作正是为了标识 GC 的工作区域，而不将他们分配到同一块连续的内存区域中是因为内存分配工作由 runtime 进行，并且内存分配都是逻辑连续的，内存布局由 runtime 维护。

*也许将它们分配到一起能提高缓存性能？（局部性原理）*
{{< /admonition >}}

最后，`makechan` 将初始化一些 `hchan` 中的字段，最终返回一个指向该 `hchan` 结构体的指针，
```go
	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	lockInit(&c.lock, lockRankHchan)
	···
	return c
```

所以最初我们使用 `myChan := make(chan Foo) // Channel without buffer` 创建一个 `chan` 时，得到的是一个指向底层 `hchan` 结构体的指针。由于在 Go 中函数都是使用值传递，因此我们向函数中传入 `chan` 指向的都是同一个底层的 `hchan` 结构体。





#### 2️⃣ 向 Channel 中写入数据
有了 channel 以后，我们就可以向 chan 中写入对应类型的元素，常见的 channel 写入语法如下所示，
```go
myChan <- Foo{}
myChanWithBuf <- Foo{}
```
上述代码段都将 `Foo{}` 实例写入到 `myChan`/`myChanWithBuf` 中，不太一样的是，`myChan` 是一个不带缓冲的 channel 而 `myChanWithBuf` 是一个带缓冲的 channel，下面我们就从源码层面对写入两种不同类型的 channel 的底层行为进行分析。具体执行向 channel 中发送元素的代码逻辑[位于](https://github.com/golang/go/blob/master/src/runtime/chan.go#L160)，其函数签名如下所示，
```go
func chansend(*hchan, unsafe.Pointer, bool, uintptr) bool
```
我们顺着代码的顺序来看看向 channel 中发送数据时，底层相应的处理逻辑，

**✨ 往 nil channel 中发送数据**
```go
	if c == nil {
		if !block {
			return false
		}
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}
```

当我们尝试向 `nil` channel 中发送数据并且可以阻塞时（block 为 1）时，该 goroutine 将会被挂起陷入永久的睡眠，并且没有机会被再次唤醒。这种情况将会造成 goroutine 泄露，即该 goroutine 所使用的资源将没办法被 runtime 回收利用。

**✨ 当非阻塞被设置时，可以执行快速失败**
```go
	// Fast path: check for failed non-blocking operation without acquiring the lock.
	//
	// After observing that the channel is not closed, we observe that the channel is
	// not ready for sending. Each of these observations is a single word-sized read
	// (first c.closed and second full()).
	// Because a closed channel cannot transition from 'ready for sending' to
	// 'not ready for sending', even if the channel is closed between the two observations,
	// they imply a moment between the two when the channel was both not yet closed
	// and not ready for sending. We behave as if we observed the channel at that moment,
	// and report that the send cannot proceed.
	//
	// It is okay if the reads are reordered here: if we observe that the channel is not
	// ready for sending and then observe that it is not closed, that implies that the
	// channel wasn't closed during the first observation. However, nothing here
	// guarantees forward progress. We rely on the side effects of lock release in
	// chanrecv() and closechan() to update this thread's view of c.closed and full().
	if !block && c.closed == 0 && full(c) {
		return false
	}
```

这一大段注释主要用来阐述为什么这个地方不需要加锁是正确的，其主要需要证明的点在于表达式的后两个条件 `c.closed == 0` 以及 `full(c)`。`c.closed` 是临界区变量，其值有可能被其它 goroutine 改变。同样的在 `full()` 中，有以下逻辑，
```go
// full reports whether a send on c would block (that is, the channel is full).
// It uses a single word-sized read of mutable state, so although
// the answer is instantaneously true, the correct answer may have changed
// by the time the calling function receives the return value.
func full(c *hchan) bool {
	// c.dataqsiz is immutable (never written after the channel is created)
	// so it is safe to read at any time during channel operation.
	if c.dataqsiz == 0 {
		// Assumes that a pointer read is relaxed-atomic.
		return c.recvq.first == nil
	}
	// Assumes that a uint read is relaxed-atomic.
	return c.qcount == c.dataqsiz
}
```

`c.dataqsiz` 在 channel 被创建时就已经决定，其代表了 channel 中缓冲区的大小。当 `c.dataqsiz == 0` 时，表明该 channel 是一个无缓冲的 channel。当接收队列中不存在等待接收的 goroutine 时，说明发送条件不满足，这时可返回 channel “满”。在有缓冲 channel 的情况下，如果当前 `buf` 循环队列已满，也可以返回 channel “满”。

这样，在 `full()` 中，`c.recvq.first` 与 `c.qcount` 位于临界区，其值也有可能在其它 goroutine 中被改变，但这不影响 `chansend` 当前发送失败的结论。原因如下，

（1）当我们观察 `c.closed == 0` 成立并且没有其它 goroutine 进入临界区的情况下，`full(c)` 成立，这种情况下 `chansend` 不满足发送条件，直接返回失败。

（2）当我们观察 `c.closed == 0` 时，有其它 goroutine 进入了临界区并在我们观察 `full(c)` 前修改了 `c.closed` 的值，让 channel 关闭，这时出现了不一致的情况。如果这时 `full(c)` 为真，也就说 channel 不满足发送的条件，这时直接返回失败是合理的。（本身向 **closed** channel 发送就是不允许的，中间改变了 `c.closed` 状态，由打开变为关闭并不会影响这个结论）。并且，改变 `c.closed` 的状态并不会让 `full()` 的判断出现异常，这就是说哪怕中间 `c.closed` 状态真的改变了，这也不会让一个 channel 由可发送的状态变为不可发送的状态（这就是说导致 `c.closed` 状态改变的动作不会对 `full()` 的结论产生影响）。

{{< admonition type=tip title="为什么不会让一个 channel 由可发送的状态变为不可发送的状态？" open=false >}}
**首先**，若创建的 channel 是一个无缓冲的 channel，channel 状态的改变并不会影响他是否是具有缓冲的。

**其次**，改变 `c.closed` 状态需要调用 `closechan()`，而 `closechan()` 并不会改变 `c.recvq.first` 与 `c.qcount` 的值（`chanrecv()` 能够改变 `c.recvq.first` 的值），而 `chanrecv()` 不能将一个可发送的 channel 变为不可发送的 channel。因为，`chanrecv()` 只会向 `recvq` 链表中插入元素或者减少 `c.qcount` 的大小。

**从而**，在执行 `full()` 前，若有 goroutine 进入了临界区并执行 `chanrecv()`，它只能将一个不可发送的 channel 变为可发送的 channel （`full()` 返回 false）。 


{{< /admonition >}}

通过无锁实现快速失败能够提高非阻塞 channel 的性能。

**✨ 接下来进行 `chansend` 的主要逻辑**

首先是对进入临界区的代码加上互斥锁进行内存同步访问控制，并且判断当前 channel 是否处于关闭状态，若当前 channel 处于关闭状态，向一个已经关闭的 channel 发送数据将会导致 **panic**，

```go
	lock(&c.lock)

	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}
```

接下来，将会尝试从接收等待队列中出列一个元素（因接收事件而被阻塞的 goroutine，由 sudog 封装），若成功取出元素，将执行 `send()` 逻辑，

```go
	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}
```

省略掉 `send()` 中相关竞态条件检测的相关逻辑，函数的主体大致如下，
```go
// send processes a send operation on an empty channel c.
// The value ep sent by the sender is copied to the receiver sg.
// The receiver is then woken up to go on its merry way.
// Channel c must be empty and locked.  send unlocks c with unlockf.
// sg must already be dequeued from c.
// ep must be non-nil and point to the heap or the caller's stack.
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	···
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
	goready(gp, skip+1)
}
```

与发送相关的逻辑在 `sendDirect()` 中，具体如下代码段所示，
```go
// Sends and receives on unbuffered or empty-buffered channels are the
// only operations where one running goroutine writes to the stack of
// another running goroutine. The GC assumes that stack writes only
// happen when the goroutine is running and are only done by that
// goroutine. Using a write barrier is sufficient to make up for
// violating that assumption, but the write barrier has to work.
// typedmemmove will call bulkBarrierPreWrite, but the target bytes
// are not in the heap, so that will not help. We arrange to call
// memmove and typeBitsBulkBarrier instead.

func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
	// src is on our stack, dst is a slot on another stack.

	// Once we read sg.elem out of sg, it will no longer
	// be updated if the destination's stack gets copied (shrunk).
	// So make sure that no preemption points can happen between read & use.
	dst := sg.elem
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
	// No need for cgo write barrier checks because dst is always
	// Go memory.
	memmove(dst, src, t.size)
}
```

介绍一下 `sendDirect()` 函数的入参，其中 `t` 是 channel 中传输元素类型的相关信息，`sg` 是当前出列的 goroutine 封装 `sudog`，`src` 是接收方的地址。

{{< admonition type=tip title="接收方地址" open=false >}}
如果我们使用 ` valSend -> myChan` 这样的语句，那么这里的 `src` 就是 `&valSend`，也即 `valSend` 变量的地址，这个地址可以指向堆上，也可以指向栈上。
{{< /admonition >}}

那么，目标地址 `dst` 自然就是接收方在 `sudog` 中留下的“收信”地址，这个地址也就是使用 `valRecv <- myChan` 语句从 channel 中接收数据时 `valRecv` 的地址，也即 `&valRecv`，我们在后面会介绍 `chanrecv()` 的相关实现。

`sendDirect()` 完成的工作就是将存放在 `src` 中的元素拷贝到 `dst` 中。细心的你可能已经发现，这个工作并没有这么简单。首先，`src` 和 `dst` 指向的位置可能是在栈上。其次，这或许是 goroutine 之间内存的拷贝。涉及一个 goroutine 向另外一个 goroutine 所持有的内存空间中进行写操作。

根据 GMP 模型，不同 goroutine 的栈是各自独有的，并且 GC 假设对栈的操作只能发生在 goroutine 正在运行中并且由当前 goroutine 来写，所以这里发送方 goroutine 向接收方 goroutine 的栈空间进行写操作可能会引发一些问题，所以在这里需要使用到写屏障来进行规避。

后续，在 `sudog` 中记录一些性能信息以及标志信息，调用 `goready` 改变当前 goroutine 状态为就绪状态，等待调度器的调度，便可以执行后续接收操作之后的代码。

若 `c.recvq` 中没有正在等待发送事件的 goroutine，并且此时的循环队列缓冲区中还有空闲的位置，此时会使用 `chanbuf` 计算当前待发送元素应该被拷贝到缓冲区的哪一个槽位的首字节内存地址，通过 `typedmemmove` 拷贝待发送元素到指定缓冲区槽位中，`c.sendx++` 指向下一个待发送元素槽位，缓冲区中元素个数加一，最后解锁并返回。

```go
	if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
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
```

如果发送逻辑走到了这一步，说明此时 channel 的缓冲区已经满了又或者是一个无缓冲的 channel，这时 goroutine 进行发送操作将会阻塞该 goroutine，构造 `sudog` 结构将该 g 挂在 `sendq` 下，并调用 `gopark` 将当前 goroutine 挂起，等待一个接收事件的发生将该 goroutine 解救出来。

```go
	// Block on the channel. Some receiver will complete our operation for us.
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)
	// Signal to anyone trying to shrink our stack that we're about
	// to park on a channel. The window between when this G's status
	// changes and when we set gp.activeStackChans is not safe for
	// stack shrinking.
	gp.parkingOnChan.Store(true)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
```

当 goroutine 被唤醒时，将会执行剩余代码进行收尾工作，

```go
	// someone woke us up.
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
	releaseSudog(mysg)
	if closed {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	return true
```

可以看到，值的复制都是在处于 running 状态的 goroutine 中完成的，哪怕这里是发送者其也不是主动进行“发送”的一方，而是由接收者代劳，从 `sudog` 中取出与发送任务相关的参数，完成相应的值复制操作（发送）。

{{< admonition type=quote title="收发数据的本质" open=false >}}
All transfer of value on the go Channels happens with the copy of value.
{{< /admonition >}}

#### 3️⃣ 向 Channel 中取出数据

#### 4️⃣ 关闭 Channel

#### 追溯 Channel 实现历史


#### 参考资料

[Go 语言设计与实现](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/)

Go 程序员面试笔试宝典：Channel 通道
